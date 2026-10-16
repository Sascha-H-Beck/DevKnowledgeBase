# Java Date & Time — Setting, Getting, and Formatting for DE, EU, USA & GB

A comprehensive guide to Java's `java.time` API covering locale-specific date/time creation, manipulation, and formatting for **Germany (DE)**, **EU**, **USA**, and **Great Britain (GB)** with best practices (Java 8+, updated through Java 21+).

---

## Table of Contents

- [Core Classes Overview](#core-classes-overview)
- [Creating Date & Time Objects](#creating-date-time-objects)
- [Getting (Accessing) Date & Time Components](#getting-accessing-date-time-components)
- [Setting (Modifying) Date & Time](#setting-modifying-date-time)
- [Time Zones for DE, EU, USA & GB](#time-zones-for-de-eu-usa--gb)
- [Formatting Patterns by Locale](#formatting-patterns-by-locale)
- [DateTimeFormatter — Locale-Specific Formatting](#datetimeformatter--locale-specific-formatting)
- [Parsing Date & Time Strings](#parsing-date-time-strings)
- [Period & Duration — Measuring Time](#period--duration--measuring-time)
- [TemporalAdjusters — Common Date Calculations](#temporaladjusters--common-date-calculations)
- [Clock — Testable Time Sources](#clock--testable-time-sources)
- [Interoperability with Legacy APIs](#interoperability-with-legacy-apis)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)
- [Locale Format Quick Reference](#locale-format-quick-reference)

---

## Core Classes Overview

All classes reside in `java.time` and are **immutable** and **thread-safe**.

| Class | Represents | Example |
| :--- | :--- | :--- |
| `LocalDate` | Date without time or zone | `2026-05-28` |
| `LocalTime` | Time without date or zone | `14:30:00` |
| `LocalDateTime` | Date and time without zone | `2026-05-28T14:30:00` |
| `ZonedDateTime` | Date, time, and time zone | `2026-05-28T14:30+02:00[Europe/Berlin]` |
| `OffsetDateTime` | Date, time, and UTC offset | `2026-05-28T14:30+02:00` |
| `Instant` | Machine timestamp (UTC epoch) | `2026-05-28T12:30:00Z` |
| `ZoneId` | Time zone identifier | `Europe/Berlin` |
| `ZoneOffset` | Fixed UTC offset | `+02:00` |
| `Duration` | Time-based amount (hours, minutes, seconds) | `PT2H30M` |
| `Period` | Date-based amount (years, months, days) | `P1Y2M3D` |

---

## Creating Date & Time Objects

### Current Date/Time

```java
LocalDate today = LocalDate.now();
LocalTime now = LocalTime.now();
LocalDateTime dateTime = LocalDateTime.now();
ZonedDateTime berlinNow = ZonedDateTime.now(ZoneId.of("Europe/Berlin"));
Instant timestamp = Instant.now();
```

### Specific Values

```java
LocalDate date = LocalDate.of(2026, 5, 28);
LocalDate date2 = LocalDate.of(2026, Month.MAY, 28);
LocalTime time = LocalTime.of(14, 30, 0);
LocalDateTime ldt = LocalDateTime.of(2026, 5, 28, 14, 30);
LocalDateTime combined = LocalDateTime.of(date, time);
```

### From Epoch

```java
LocalDate fromEpoch = LocalDate.ofEpochDay(20601);
Instant fromMillis = Instant.ofEpochMilli(1780070400000L);
Instant fromSeconds = Instant.ofEpochSecond(1780070400L);
```

### Zone-Aware Creation

```java
ZonedDateTime berlinTime = ZonedDateTime.of(
    LocalDateTime.of(2026, 5, 28, 14, 30),
    ZoneId.of("Europe/Berlin")
);

ZonedDateTime londonTime = ZonedDateTime.of(
    LocalDateTime.of(2026, 5, 28, 13, 30),
    ZoneId.of("Europe/London")
);

ZonedDateTime newYorkTime = ZonedDateTime.of(
    LocalDateTime.of(2026, 5, 28, 8, 30),
    ZoneId.of("America/New_York")
);
```

---

## Getting (Accessing) Date & Time Components

### Date Components

| Method | Returns | Example for `2026-05-28` |
| :--- | :--- | :--- |
| `getYear()` | `int` | `2026` |
| `getMonth()` | `Month` | `MAY` |
| `getMonthValue()` | `int` | `5` |
| `getDayOfMonth()` | `int` | `28` |
| `getDayOfWeek()` | `DayOfWeek` | `THURSDAY` |
| `getDayOfYear()` | `int` | `148` |
| `lengthOfMonth()` | `int` | `31` |
| `lengthOfYear()` | `int` | `365` |
| `isLeapYear()` | `boolean` | `false` |

### Time Components

| Method | Returns | Example for `14:30:45.123456789` |
| :--- | :--- | :--- |
| `getHour()` | `int` | `14` |
| `getMinute()` | `int` | `30` |
| `getSecond()` | `int` | `45` |
| `getNano()` | `int` | `123456789` |

### Zone Components (ZonedDateTime)

| Method | Returns | Example for Berlin in summer |
| :--- | :--- | :--- |
| `getZone()` | `ZoneId` | `Europe/Berlin` |
| `getOffset()` | `ZoneOffset` | `+02:00` |
| `toLocalDate()` | `LocalDate` | `2026-05-28` |
| `toLocalTime()` | `LocalTime` | `14:30:00` |
| `toInstant()` | `Instant` | `2026-05-28T12:30:00Z` |

### Generic Field Access (TemporalField)

```java
int weekOfYear = date.get(ChronoField.ALIGNED_WEEK_OF_YEAR);
int quarter = date.get(IsoFields.QUARTER_OF_YEAR);
long epochDay = date.getLong(ChronoField.EPOCH_DAY);
```

---

## Setting (Modifying) Date & Time

All `java.time` classes are immutable — "setting" always returns a new instance.

### with* Methods (Absolute Setting)

```java
LocalDate date = LocalDate.of(2026, 5, 28);
LocalDate newYear = date.withYear(2027);           // 2027-05-28
LocalDate newMonth = date.withMonth(12);           // 2026-12-28
LocalDate newDay = date.withDayOfMonth(1);         // 2026-05-01
LocalDate dayOfYear = date.withDayOfYear(1);       // 2026-01-01

LocalTime time = LocalTime.of(14, 30);
LocalTime newHour = time.withHour(9);              // 09:30
LocalTime newMinute = time.withMinute(0);          // 14:00
LocalTime newSecond = time.withSecond(59);         // 14:30:59
```

### plus/minus Methods (Relative Setting)

```java
LocalDate tomorrow = today.plusDays(1);
LocalDate nextWeek = today.plusWeeks(1);
LocalDate nextMonth = today.plusMonths(1);
LocalDate nextYear = today.plusYears(1);

LocalTime later = now.plusHours(2).plusMinutes(30);
LocalTime earlier = now.minusHours(1);

LocalDateTime future = dateTime.plusDays(7).plusHours(3);
```

### with(TemporalField) — Generic Setting

```java
LocalDate adjusted = date.with(ChronoField.DAY_OF_WEEK, 1); // Monday of same week
LocalDate q3Start = date.with(IsoFields.DAY_OF_QUARTER, 1); // first day of quarter
```

---

## Time Zones for DE, EU, USA & GB

### Zone IDs

| Region | ZoneId | UTC Offset (Winter) | UTC Offset (Summer/DST) |
| :--- | :--- | :--- | :--- |
| Germany | `Europe/Berlin` | +01:00 (CET) | +02:00 (CEST) |
| EU Central | `Europe/Paris`, `Europe/Rome`, `Europe/Madrid` | +01:00 (CET) | +02:00 (CEST) |
| EU Eastern | `Europe/Athens`, `Europe/Helsinki` | +02:00 (EET) | +03:00 (EEST) |
| EU Western | `Europe/Lisbon` | +00:00 (WET) | +01:00 (WEST) |
| Great Britain | `Europe/London` | +00:00 (GMT) | +01:00 (BST) |
| USA Eastern | `America/New_York` | -05:00 (EST) | -04:00 (EDT) |
| USA Central | `America/Chicago` | -06:00 (CST) | -05:00 (CDT) |
| USA Mountain | `America/Denver` | -07:00 (MST) | -06:00 (MDT) |
| USA Pacific | `America/Los_Angeles` | -08:00 (PST) | -07:00 (PDT) |

### Zone Conversion

```java
ZonedDateTime berlin = ZonedDateTime.now(ZoneId.of("Europe/Berlin"));
ZonedDateTime london = berlin.withZoneSameInstant(ZoneId.of("Europe/London"));
ZonedDateTime newYork = berlin.withZoneSameInstant(ZoneId.of("America/New_York"));
ZonedDateTime losAngeles = berlin.withZoneSameInstant(ZoneId.of("America/Los_Angeles"));

// Same wall-clock time, different zone (changes the instant!)
ZonedDateTime sameLocal = berlin.withZoneSameLocal(ZoneId.of("America/New_York"));
```

### DST-Safe Arithmetic

```java
// Handles DST transitions correctly
ZonedDateTime beforeDST = ZonedDateTime.of(
    LocalDateTime.of(2026, 3, 29, 1, 30),
    ZoneId.of("Europe/Berlin")
);
ZonedDateTime afterDST = beforeDST.plusHours(1); // jumps to 03:30 (CET → CEST)
```

---

## Formatting Patterns by Locale

### Standard Date Formats by Region

| Region | Short | Medium | Long |
| :--- | :--- | :--- | :--- |
| Germany (DE) | `28.05.26` | `28.05.2026` | `28. Mai 2026` |
| EU (FR example) | `28/05/26` | `28/05/2026` | `28 mai 2026` |
| USA | `5/28/26` | `May 28, 2026` | `May 28, 2026` |
| Great Britain | `28/05/26` | `28 May 2026` | `28 May 2026` |

### Standard Time Formats by Region

| Region | Short | Medium |
| :--- | :--- | :--- |
| Germany (DE) | `14:30` | `14:30:00` |
| EU (FR) | `14:30` | `14:30:00` |
| USA | `2:30 PM` | `2:30:00 PM` |
| Great Britain | `14:30` | `14:30:00` |

### Custom Pattern Letters

| Symbol | Meaning | Example |
| :--- | :--- | :--- |
| `y` | Year | `2026` |
| `M` | Month (numeric/text) | `5`, `05`, `May`, `Mai` |
| `d` | Day of month | `28` |
| `E` | Day of week (text) | `Thu`, `Donnerstag` |
| `H` | Hour (0-23) | `14` |
| `h` | Hour (1-12) | `2` |
| `m` | Minute | `30` |
| `s` | Second | `45` |
| `a` | AM/PM | `PM`, `nachm.` |
| `z` | Time zone name | `CEST`, `BST` |
| `Z` | Zone offset | `+0200` |
| `X` | Zone offset (ISO) | `+02:00`, `Z` |
| `V` | Zone ID | `Europe/Berlin` |
| `G` | Era | `AD`, `n. Chr.` |
| `'...'` | Literal text | `'at'` → `at` |

---

## DateTimeFormatter — Locale-Specific Formatting

### Predefined Formatters

| Formatter | Output Example |
| :--- | :--- |
| `ISO_LOCAL_DATE` | `2026-05-28` |
| `ISO_LOCAL_TIME` | `14:30:00` |
| `ISO_LOCAL_DATE_TIME` | `2026-05-28T14:30:00` |
| `ISO_ZONED_DATE_TIME` | `2026-05-28T14:30:00+02:00[Europe/Berlin]` |
| `ISO_INSTANT` | `2026-05-28T12:30:00Z` |
| `ISO_OFFSET_DATE_TIME` | `2026-05-28T14:30:00+02:00` |
| `RFC_1123_DATE_TIME` | `Thu, 28 May 2026 14:30:00 +0200` |

### Locale-Based Formatting

```java
LocalDate date = LocalDate.of(2026, 5, 28);
LocalTime time = LocalTime.of(14, 30, 0);
LocalDateTime dateTime = LocalDateTime.of(date, time);

// Germany
DateTimeFormatter deShort = DateTimeFormatter.ofLocalizedDate(FormatStyle.SHORT)
    .withLocale(Locale.GERMANY);
DateTimeFormatter deMedium = DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM)
    .withLocale(Locale.GERMANY);
DateTimeFormatter deLong = DateTimeFormatter.ofLocalizedDate(FormatStyle.LONG)
    .withLocale(Locale.GERMANY);
DateTimeFormatter deFull = DateTimeFormatter.ofLocalizedDate(FormatStyle.FULL)
    .withLocale(Locale.GERMANY);

date.format(deShort);   // "28.05.26"
date.format(deMedium);  // "28.05.2026"
date.format(deLong);    // "28. Mai 2026"
date.format(deFull);    // "Donnerstag, 28. Mai 2026"

// Great Britain
DateTimeFormatter gbMedium = DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM)
    .withLocale(Locale.UK);
date.format(gbMedium);  // "28 May 2026"

// USA
DateTimeFormatter usMedium = DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM)
    .withLocale(Locale.US);
date.format(usMedium);  // "May 28, 2026"

// Date and Time together
DateTimeFormatter deDateTime = DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM)
    .withLocale(Locale.GERMANY);
dateTime.format(deDateTime);  // "28.05.2026, 14:30:00"
```

### Custom Patterns with Locale

```java
// German custom format
DateTimeFormatter deCustom = DateTimeFormatter.ofPattern("dd. MMMM yyyy, HH:mm 'Uhr'", Locale.GERMANY);
dateTime.format(deCustom);  // "28. Mai 2026, 14:30 Uhr"

// GB custom format
DateTimeFormatter gbCustom = DateTimeFormatter.ofPattern("d MMMM yyyy, HH:mm", Locale.UK);
dateTime.format(gbCustom);  // "28 May 2026, 14:30"

// US custom format
DateTimeFormatter usCustom = DateTimeFormatter.ofPattern("MMMM d, yyyy 'at' h:mm a", Locale.US);
dateTime.format(usCustom);  // "May 28, 2026 at 2:30 PM"

// EU/French custom format
DateTimeFormatter frCustom = DateTimeFormatter.ofPattern("d MMMM yyyy 'à' HH'h'mm", Locale.FRANCE);
dateTime.format(frCustom);  // "28 mai 2026 à 14h30"
```

### Zone-Aware Formatting

```java
ZonedDateTime zdt = ZonedDateTime.of(dateTime, ZoneId.of("Europe/Berlin"));

DateTimeFormatter withZone = DateTimeFormatter.ofPattern(
    "dd.MM.yyyy HH:mm z (VV)", Locale.GERMANY
);
zdt.format(withZone);  // "28.05.2026 14:30 MESZ (Europe/Berlin)"

DateTimeFormatter usWithZone = DateTimeFormatter.ofPattern(
    "MM/dd/yyyy hh:mm a z", Locale.US
);
ZonedDateTime usZdt = zdt.withZoneSameInstant(ZoneId.of("America/New_York"));
usZdt.format(usWithZone);  // "05/28/2026 08:30 AM EDT"
```

---

## Parsing Date & Time Strings

### Basic Parsing

```java
LocalDate date = LocalDate.parse("2026-05-28");
LocalTime time = LocalTime.parse("14:30:00");
LocalDateTime dateTime = LocalDateTime.parse("2026-05-28T14:30:00");
ZonedDateTime zdt = ZonedDateTime.parse("2026-05-28T14:30:00+02:00[Europe/Berlin]");
```

### Locale-Specific Parsing

```java
// German date format
DateTimeFormatter deFormat = DateTimeFormatter.ofPattern("dd.MM.yyyy", Locale.GERMANY);
LocalDate deDate = LocalDate.parse("28.05.2026", deFormat);

// US date format
DateTimeFormatter usFormat = DateTimeFormatter.ofPattern("MM/dd/yyyy", Locale.US);
LocalDate usDate = LocalDate.parse("05/28/2026", usFormat);

// GB date format
DateTimeFormatter gbFormat = DateTimeFormatter.ofPattern("dd/MM/yyyy", Locale.UK);
LocalDate gbDate = LocalDate.parse("28/05/2026", gbFormat);

// German date with text month
DateTimeFormatter deTextFormat = DateTimeFormatter.ofPattern("d. MMMM yyyy", Locale.GERMANY);
LocalDate deTextDate = LocalDate.parse("28. Mai 2026", deTextFormat);

// US with AM/PM
DateTimeFormatter usTimeFormat = DateTimeFormatter.ofPattern("MM/dd/yyyy hh:mm a", Locale.US);
LocalDateTime usDateTime = LocalDateTime.parse("05/28/2026 02:30 PM", usTimeFormat);
```

### Lenient and Strict Parsing

```java
// Strict (default) — rejects ambiguous input
DateTimeFormatter strict = DateTimeFormatter.ofPattern("dd.MM.yyyy")
    .withResolverStyle(ResolverStyle.STRICT);

// Lenient — adjusts invalid dates (e.g., Feb 30 → Feb 28)
DateTimeFormatter lenient = DateTimeFormatter.ofPattern("dd.MM.yyyy")
    .withResolverStyle(ResolverStyle.LENIENT);

// Smart (default for ofPattern) — adjusts within reason
DateTimeFormatter smart = DateTimeFormatter.ofPattern("dd.MM.yyyy")
    .withResolverStyle(ResolverStyle.SMART);
```

### Optional Sections in Patterns

```java
// Parse time with or without seconds
DateTimeFormatter flexTime = DateTimeFormatter.ofPattern("HH:mm[:ss]");
LocalTime t1 = LocalTime.parse("14:30", flexTime);     // 14:30:00
LocalTime t2 = LocalTime.parse("14:30:45", flexTime);  // 14:30:45
```

---

## Period & Duration — Measuring Time

### Period (Date-Based)

```java
Period oneYear = Period.ofYears(1);
Period twoMonths = Period.ofMonths(2);
Period tenDays = Period.ofDays(10);
Period complex = Period.of(1, 2, 3); // 1 year, 2 months, 3 days

// Between two dates
Period between = Period.between(
    LocalDate.of(2026, 1, 1),
    LocalDate.of(2026, 5, 28)
);
between.getMonths(); // 4
between.getDays();   // 27

// Apply to date
LocalDate future = today.plus(Period.ofMonths(3));
```

### Duration (Time-Based)

```java
Duration twoHours = Duration.ofHours(2);
Duration thirtyMinutes = Duration.ofMinutes(30);
Duration fiveSeconds = Duration.ofSeconds(5);
Duration halfSecond = Duration.ofMillis(500);

// Between two times
Duration between = Duration.between(
    LocalTime.of(9, 0),
    LocalTime.of(17, 30)
);
between.toHours();   // 8
between.toMinutes(); // 510

// Apply to time
LocalTime endOfDay = now.plus(Duration.ofHours(8));
```

### ChronoUnit for Measuring

```java
long daysBetween = ChronoUnit.DAYS.between(startDate, endDate);
long hoursBetween = ChronoUnit.HOURS.between(startTime, endTime);
long monthsBetween = ChronoUnit.MONTHS.between(startDate, endDate);
```

---

## TemporalAdjusters — Common Date Calculations

| Adjuster | Description |
| :--- | :--- |
| `firstDayOfMonth()` | First day of the current month |
| `lastDayOfMonth()` | Last day of the current month |
| `firstDayOfNextMonth()` | First day of the next month |
| `firstDayOfYear()` | First day of the current year |
| `lastDayOfYear()` | Last day of the current year |
| `firstDayOfNextYear()` | First day of the next year |
| `next(DayOfWeek)` | Next occurrence of the given day |
| `nextOrSame(DayOfWeek)` | Next or same day if already that day |
| `previous(DayOfWeek)` | Previous occurrence of the given day |
| `previousOrSame(DayOfWeek)` | Previous or same day |
| `firstInMonth(DayOfWeek)` | First specific day of current month |
| `lastInMonth(DayOfWeek)` | Last specific day of current month |
| `dayOfWeekInMonth(n, DayOfWeek)` | Nth specific day of current month |

```java
LocalDate today = LocalDate.of(2026, 5, 28);

LocalDate endOfMonth = today.with(TemporalAdjusters.lastDayOfMonth());       // 2026-05-31
LocalDate nextMonday = today.with(TemporalAdjusters.next(DayOfWeek.MONDAY)); // 2026-06-01
LocalDate firstOfYear = today.with(TemporalAdjusters.firstDayOfYear());      // 2026-01-01

// Third Friday of the month
LocalDate thirdFriday = today.with(
    TemporalAdjusters.dayOfWeekInMonth(3, DayOfWeek.FRIDAY)
);

// Custom adjuster
TemporalAdjuster nextWorkday = temporal -> {
    LocalDate d = LocalDate.from(temporal);
    do { d = d.plusDays(1); }
    while (d.getDayOfWeek() == DayOfWeek.SATURDAY ||
           d.getDayOfWeek() == DayOfWeek.SUNDAY);
    return d;
};
LocalDate nextWork = today.with(nextWorkday);
```

---

## Clock — Testable Time Sources

```java
// System clock (default behaviour)
Clock systemClock = Clock.systemDefaultZone();
Clock berlinClock = Clock.system(ZoneId.of("Europe/Berlin"));
Clock utcClock = Clock.systemUTC();

// Fixed clock for testing
Clock fixed = Clock.fixed(
    Instant.parse("2026-05-28T12:00:00Z"),
    ZoneId.of("Europe/Berlin")
);
LocalDate.now(fixed); // always 2026-05-28

// Offset clock for simulation
Clock twoHoursAhead = Clock.offset(Clock.systemUTC(), Duration.ofHours(2));

// Inject into production code
public class OrderService {
    private final Clock clock;

    public OrderService(Clock clock) {
        this.clock = clock;
    }

    public LocalDateTime getOrderTimestamp() {
        return LocalDateTime.now(clock);
    }
}
```

---

## Interoperability with Legacy APIs

### java.util.Date ↔ java.time

```java
// Date → Instant → ZonedDateTime/LocalDateTime
Date legacyDate = new Date();
Instant instant = legacyDate.toInstant();
ZonedDateTime zdt = instant.atZone(ZoneId.of("Europe/Berlin"));
LocalDateTime ldt = LocalDateTime.ofInstant(instant, ZoneId.of("Europe/Berlin"));

// LocalDateTime → Instant → Date
LocalDateTime now = LocalDateTime.now();
Instant inst = now.atZone(ZoneId.of("Europe/Berlin")).toInstant();
Date date = Date.from(inst);
```

### java.util.Calendar ↔ java.time

```java
// Calendar → ZonedDateTime
Calendar cal = Calendar.getInstance(TimeZone.getTimeZone("Europe/Berlin"));
ZonedDateTime zdt = cal.toInstant().atZone(cal.getTimeZone().toZoneId());

// ZonedDateTime → Calendar
ZonedDateTime zdt2 = ZonedDateTime.now(ZoneId.of("Europe/Berlin"));
GregorianCalendar cal2 = GregorianCalendar.from(zdt2);
```

### java.sql Types ↔ java.time

```java
// java.sql.Date ↔ LocalDate
java.sql.Date sqlDate = java.sql.Date.valueOf(LocalDate.now());
LocalDate localDate = sqlDate.toLocalDate();

// java.sql.Timestamp ↔ LocalDateTime / Instant
java.sql.Timestamp sqlTs = java.sql.Timestamp.valueOf(LocalDateTime.now());
LocalDateTime fromTs = sqlTs.toLocalDateTime();
Instant fromTsInstant = sqlTs.toInstant();
```

---

## Best Practices

### 1. Always Use java.time (Never java.util.Date/Calendar)

The legacy `Date` and `Calendar` classes are mutable, poorly designed, and error-prone. Always prefer `java.time`.

### 2. Store Timestamps as Instant or UTC

```java
// Store instants in UTC for databases and serialization
Instant created = Instant.now();

// Convert to local only for display
ZonedDateTime display = created.atZone(ZoneId.of("Europe/Berlin"));
```

### 3. Use ZoneId, Not Fixed Offsets

```java
// Correct — handles DST automatically
ZoneId berlin = ZoneId.of("Europe/Berlin");

// Avoid — ignores DST transitions
ZoneOffset fixed = ZoneOffset.ofHours(2);
```

### 4. Choose the Right Type

| Use Case | Type |
| :--- | :--- |
| Birthday, holiday | `LocalDate` |
| Daily alarm | `LocalTime` |
| Appointment (user's local) | `LocalDateTime` |
| Scheduled event across zones | `ZonedDateTime` |
| Database timestamp, audit log | `Instant` |
| API exchange, serialization | `OffsetDateTime` or `Instant` |

### 5. Reuse Formatters (They Are Thread-Safe)

```java
// Define once, use everywhere
public final class Formats {
    public static final DateTimeFormatter DE_DATE =
        DateTimeFormatter.ofPattern("dd.MM.yyyy", Locale.GERMANY);

    public static final DateTimeFormatter DE_DATETIME =
        DateTimeFormatter.ofPattern("dd.MM.yyyy HH:mm", Locale.GERMANY);

    public static final DateTimeFormatter US_DATE =
        DateTimeFormatter.ofPattern("MM/dd/yyyy", Locale.US);

    public static final DateTimeFormatter GB_DATE =
        DateTimeFormatter.ofPattern("dd/MM/yyyy", Locale.UK);

    private Formats() {}
}
```

### 6. Use Locale Constants

| Locale | Constant |
| :--- | :--- |
| Germany | `Locale.GERMANY` |
| France | `Locale.FRANCE` |
| Italy | `Locale.ITALY` |
| UK | `Locale.UK` |
| US | `Locale.US` |
| Custom | `Locale.of("de", "AT")` (Austria) |

### 7. Inject Clock for Testability

Never call `LocalDateTime.now()` directly in business logic. Inject a `Clock` to make code testable.

### 8. Handle DST Gaps and Overlaps

```java
// During spring-forward gap (e.g., 02:30 does not exist in Europe/Berlin on DST switch)
ZonedDateTime adjusted = ZonedDateTime.of(
    LocalDateTime.of(2026, 3, 29, 2, 30), // 02:30 does not exist
    ZoneId.of("Europe/Berlin")
); // Automatically adjusts to 03:30 CEST
```

---

## Common Pitfalls

### Pitfall: Month Indexing

```java
// java.util.Calendar months are 0-based (January = 0)
// java.time months are 1-based (January = 1) — CORRECT!
LocalDate.of(2026, 1, 1); // January 1st
```

### Pitfall: LocalDateTime Has No Zone

```java
// This is just a date/time — it does NOT represent a unique instant
LocalDateTime ldt = LocalDateTime.of(2026, 5, 28, 14, 30);
// "14:30 on May 28" could mean different instants in Berlin vs. New York
```

### Pitfall: Ignoring Return Values

```java
// WRONG — immutable objects, original unchanged
date.plusDays(1);

// CORRECT
LocalDate tomorrow = date.plusDays(1);
```

### Pitfall: Parsing AM/PM Without Locale

```java
// May fail on non-English JVMs
DateTimeFormatter bad = DateTimeFormatter.ofPattern("hh:mm a");

// Correct — explicit locale
DateTimeFormatter good = DateTimeFormatter.ofPattern("hh:mm a", Locale.US);
```

### Pitfall: Using ZoneOffset Instead of ZoneId

```java
// WRONG — does not handle DST changes
OffsetDateTime odt = OffsetDateTime.now(ZoneOffset.ofHours(1));

// CORRECT — adjusts for CET/CEST automatically
ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("Europe/Berlin"));
```

---

## Locale Format Quick Reference

### Complete Formatting Examples

```java
LocalDateTime dt = LocalDateTime.of(2026, 5, 28, 14, 30, 0);
ZonedDateTime zdtBerlin = dt.atZone(ZoneId.of("Europe/Berlin"));

// --- GERMANY ---
DateTimeFormatter.ofLocalizedDateTime(FormatStyle.SHORT).withLocale(Locale.GERMANY)
    // → "28.05.26, 14:30"
DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM).withLocale(Locale.GERMANY)
    // → "28.05.2026, 14:30:00"
DateTimeFormatter.ofPattern("EEEE, dd. MMMM yyyy 'um' HH:mm 'Uhr'", Locale.GERMANY)
    // → "Donnerstag, 28. Mai 2026 um 14:30 Uhr"

// --- USA ---
DateTimeFormatter.ofLocalizedDateTime(FormatStyle.SHORT).withLocale(Locale.US)
    // → "5/28/26, 2:30 PM"
DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM).withLocale(Locale.US)
    // → "May 28, 2026, 2:30:00 PM"
DateTimeFormatter.ofPattern("EEEE, MMMM d, yyyy 'at' h:mm a", Locale.US)
    // → "Thursday, May 28, 2026 at 2:30 PM"

// --- GREAT BRITAIN ---
DateTimeFormatter.ofLocalizedDateTime(FormatStyle.SHORT).withLocale(Locale.UK)
    // → "28/05/2026, 14:30"
DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM).withLocale(Locale.UK)
    // → "28 May 2026, 14:30:00"
DateTimeFormatter.ofPattern("EEEE, d MMMM yyyy, HH:mm", Locale.UK)
    // → "Thursday, 28 May 2026, 14:30"

// --- FRANCE (EU example) ---
DateTimeFormatter.ofLocalizedDateTime(FormatStyle.SHORT).withLocale(Locale.FRANCE)
    // → "28/05/2026 14:30"
DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM).withLocale(Locale.FRANCE)
    // → "28 mai 2026, 14:30:00"
DateTimeFormatter.ofPattern("EEEE d MMMM yyyy 'à' HH'h'mm", Locale.FRANCE)
    // → "jeudi 28 mai 2026 à 14h30"
```

### ISO Standard Formats (Locale-Independent)

```java
LocalDate date = LocalDate.of(2026, 5, 28);
Instant instant = Instant.parse("2026-05-28T12:30:00Z");

date.format(DateTimeFormatter.ISO_DATE);              // "2026-05-28"
instant.toString();                                    // "2026-05-28T12:30:00Z"
OffsetDateTime.now().format(DateTimeFormatter.ISO_OFFSET_DATE_TIME);
    // "2026-05-28T14:30:00+02:00"
```

### Week Numbering Differences

```java
// ISO week (used in DE, EU, GB): week starts Monday, first week has >= 4 days in new year
WeekFields isoWeek = WeekFields.ISO;
int isoWeekNum = date.get(isoWeek.weekOfWeekBasedYear());

// US week: week starts Sunday, first week contains January 1
WeekFields usWeek = WeekFields.of(Locale.US);
int usWeekNum = date.get(usWeek.weekOfWeekBasedYear());
```
