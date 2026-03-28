---
layout: post
title: "Scala for Data Pipelines: Lessons from Analytics at Scale"
comments: true
description: "Using Scala and Spark for user analytics at Indus OS, processing millions of events from budget Android phones on 2G networks"
keywords: "scala, spark, data pipelines, analytics, android, indus os, rdd, case classes"
---

Indus OS had 8 million users on budget Android phones. Most were on 2G networks in tier-2 and tier-3 Indian cities. We needed analytics to understand app usage, language preferences, and feature adoption. The challenge: telemetry from these devices was sparse, delayed, and often malformed.

## The Data Problem

Events arrived hours or days late. A user on 2G might generate events at 10am but the phone would batch and send them at 2am when connected to WiFi. Some events arrived with timestamps in the future (clock drift on cheap phones). About 12% of events had missing or corrupted fields.

We modeled events as sealed case class hierarchies. This let the compiler catch missing pattern matches and made the pipeline self-documenting.

```scala
sealed trait UserEvent {
  def userId: String
  def timestamp: Long
  def deviceId: String
}

case class AppLaunch(
  userId: String, timestamp: Long, deviceId: String,
  packageName: String, sessionDurationMs: Long
) extends UserEvent

case class LanguageSwitch(
  userId: String, timestamp: Long, deviceId: String,
  fromLang: String, toLang: String
) extends UserEvent

case class TtsUsage(
  userId: String, timestamp: Long, deviceId: String,
  language: String, utteranceLengthMs: Long
) extends UserEvent
```

## Building Fault-Tolerant Pipelines with Spark

We processed ~15 million events per day on a 4-node Spark cluster. Pattern matching on the sealed trait made event routing clean and exhaustive.

```scala
def processEvents(events: RDD[UserEvent]): Unit = {
  val cleaned = events
    .filter(e => e.timestamp > 0 && e.timestamp < System.currentTimeMillis() + 86400000)
    .filter(e => e.userId.nonEmpty && e.deviceId.nonEmpty)

  // Route by event type
  cleaned.foreach {
    case e: AppLaunch      => recordAppSession(e)
    case e: LanguageSwitch => trackLanguageAdoption(e)
    case e: TtsUsage       => measureTtsEngagement(e)
  }
}

def dailyLanguageStats(events: RDD[UserEvent]): RDD[(String, Long)] = {
  events.collect { case e: LanguageSwitch => e }
    .map(e => (e.toLang, 1L))
    .reduceByKey(_ + _)
    .sortBy(_._2, ascending = false)
}
```

For late-arriving data, we used a two-pass approach. The first pass processed events within a 24-hour window. A second pass ran 48 hours later to catch stragglers and reconcile counts. This recovered about 8% of events that would otherwise be lost.

## Practical Scala Patterns That Helped

`Option` and `Try` were essential for handling the messy telemetry data. Instead of null checks scattered everywhere, we parsed raw JSON into `Option` fields and let the pipeline gracefully skip bad records.

```scala
def parseEvent(raw: String): Option[UserEvent] = {
  Try {
    val j = parse(raw)
    val uid = (j \ "uid").extract[String]
    val ts  = (j \ "ts").extract[Long]
    val did = (j \ "did").extract[String]
    (j \ "type").extract[String] match {
      case "app_launch"  => AppLaunch(uid, ts, did,
        (j \ "pkg").extract[String], (j \ "dur").extract[Long])
      case "lang_switch" => LanguageSwitch(uid, ts, did,
        (j \ "from").extract[String], (j \ "to").extract[String])
      case other => throw new IllegalArgumentException(s"Unknown: $other")
    }
  }.toOption
}

// Bad records silently dropped, counted per batch
val parsed: RDD[UserEvent] = rawEvents.flatMap(parseEvent)
```

We logged dropped event counts per batch. On a typical day, 3-4% of events failed parsing. Most were from older app versions sending a deprecated schema.

## What Worked and What Didn't

Scala's type system caught entire categories of bugs at compile time. When we added a new event type, the compiler flagged every incomplete pattern match. This saved us multiple times during the 6 months I worked on this pipeline.

What didn't work: we initially tried Spark Streaming for real-time analytics. On 2G networks, events arrived in unpredictable bursts. The micro-batch model created either empty batches or massive spikes. We switched to hourly batch jobs which were simpler and matched the actual data arrival pattern. Sometimes batch is the right answer.
