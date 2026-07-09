# SQL commands and testing information

## Introduction

This is a collect of information on useful SQL commands for looking at the database and for using with testing for correct functioning. It can allow for loading of special data to see how it functions. It also shows SQL commands that my be interesting and useful in other contexts and modifiable.

It is provided for the record and for potential use. Some items may be out of date or have issues as they have not been retested. Some assume the basic test data has been loaded.

This is only the first version with limited information. More should be coming in the future. If you have ideas for additions/changes then please make a pull request.

## Wiping OED DB data

### These should work after time-varying changes

- Remove all readings, groups, meters, conversions, units and the cik data.

```sql
delete from readings;
delete from groups_immediate_meters;
delete from groups_immediate_children;
delete from groups;
delete from meters;
delete from conversion_segments;
delete from conversions;
delete from cik;
delete from cik_vary;
delete from units;
```

- Purge suffix units. Works after time-varying work.

```sql
DELETE FROM CONVERSION_SEGMENTS WHERE DESTINATION_ID IN ( SELECT ID FROM UNITS WHERE SUFFIX != '' OR TYPE_OF_UNIT = 'suffix' );
DELETE FROM CONVERSION_SEGMENTS WHERE SOURCE_ID IN ( SELECT ID FROM UNITS WHERE SUFFIX != '' OR TYPE_OF_UNIT = 'suffix' );
DELETE FROM CONVERSIONS WHERE DESTINATION_ID IN ( SELECT ID FROM UNITS WHERE SUFFIX != '' OR TYPE_OF_UNIT = 'suffix' );
DELETE FROM CONVERSIONS WHERE SOURCE_ID IN ( SELECT ID FROM UNITS WHERE SUFFIX != '' OR TYPE_OF_UNIT = 'suffix' );
DELETE FROM CIK;
DELETE FROM READINGS WHERE METER_ID IN ( SELECT ID FROM METERS WHERE DEFAULT_GRAPHIC_UNIT IN ( SELECT ID FROM UNITS WHERE SUFFIX != '' OR TYPE_OF_UNIT = 'suffix' ) );
DELETE FROM METERS WHERE DEFAULT_GRAPHIC_UNIT IN ( SELECT ID FROM UNITS WHERE SUFFIX != '' OR TYPE_OF_UNIT = 'suffix' );
DELETE FROM UNITS WHERE SUFFIX != '' OR TYPE_OF_UNIT = 'suffix';
```

- Remove the readings, meters/groups, units & ciks associated with the Water Gallon/gallon test data.

```sql
DELETE FROM READINGS WHERE METER_ID NOT IN ( SELECT ID FROM METERS WHERE NAME = 'Water Gallon');
delete from groups_immediate_meters;
delete from groups_immediate_children;
delete from groups;
DELETE FROM METERS WHERE ID NOT IN ( SELECT ID FROM METERS WHERE NAME = 'Water Gallon');
DELETE FROM CIK WHERE SOURCE_ID NOT IN ( SELECT UNIT_ID FROM METERS WHERE NAME = 'Water Gallon' ) OR DESTINATION_ID NOT IN ( SELECT ID FROM UNITS WHERE NAME = 'gallon' );
DELETE FROM CIK_VARY WHERE SOURCE_ID NOT IN ( SELECT UNIT_ID FROM METERS WHERE NAME = 'Water Gallon' ) OR DESTINATION_ID NOT IN ( SELECT ID FROM UNITS WHERE NAME = 'gallon' );
DELETE FROM UNITS WHERE ID NOT IN ( SELECT UNIT_ID FROM METERS WHERE NAME = 'Water Gallon' ) AND ID NOT IN ( SELECT ID FROM UNITS WHERE NAME = 'gallon' );
```

- Remove all units, conversions, ... but the water ones. Probably easier to delete them all and just recreate ones wanted.

```sql
DELETE FROM CONVERSION_SEGMENTS WHERE SOURCE_ID NOT IN ( SELECT UNIT_ID FROM METERS WHERE NAME = 'Water Gallon' ) OR DESTINATION_ID NOT IN ( SELECT ID FROM UNITS WHERE NAME = 'gallon' );
DELETE FROM CONVERSIONS WHERE SOURCE_ID NOT IN ( SELECT UNIT_ID FROM METERS WHERE NAME = 'Water Gallon' ) OR DESTINATION_ID NOT IN ( SELECT ID FROM UNITS WHERE NAME = 'gallon' );
-- I made a mistake and did METER_UNIT for ID select so had to remove all readings since ones wanted were gone.
DELETE FROM READINGS WHERE METER_ID NOT IN ( SELECT ID FROM METERS WHERE NAME = 'Water Gallon');
delete from groups_immediate_meters;
delete from groups_immediate_children;
delete from groups;
DELETE FROM METERS WHERE ID NOT IN ( SELECT ID FROM METERS WHERE NAME = 'Water Gallon');
DELETE FROM CIK WHERE SOURCE_ID NOT IN ( SELECT UNIT_ID FROM METERS WHERE NAME = 'Water Gallon' ) OR DESTINATION_ID NOT IN ( SELECT ID FROM UNITS WHERE NAME = 'gallon' );
DELETE FROM CIK_VARY WHERE SOURCE_ID NOT IN ( SELECT UNIT_ID FROM METERS WHERE NAME = 'Water Gallon' ) OR DESTINATION_ID NOT IN ( SELECT ID FROM UNITS WHERE NAME = 'gallon' );
DELETE FROM UNITS WHERE ID NOT IN ( SELECT UNIT_ID FROM METERS WHERE NAME = 'Water Gallon' ) AND ID NOT IN ( SELECT ID FROM UNITS WHERE NAME = 'gallon' );
```

- Make water (see previous example) have 2 segments (no pattern) where removal all other ones if they exist:

```sql
DELETE FROM CIK;
DELETE FROM CIK_VARY;
DELETE FROM CONVERSION_SEGMENTS;
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT UNIT_ID FROM METERS WHERE NAME = 'Water Gallon' ), ( SELECT ID FROM UNITS WHERE NAME = 'gallon' ), NULL, 2, 0, '-infinity', '2021-06-03 00:00:00', 'segment 1 Water Gallon -> gallon' );
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT UNIT_ID FROM METERS WHERE NAME = 'Water Gallon' ), ( SELECT ID FROM UNITS WHERE NAME = 'gallon' ), NULL, 3, 0, '2021-06-03 00:00:00', 'infinity', 'segment 2 Water Gallon -> gallon' );
```

## Time varying

- Change the end time of the one segment of the test data for Water Gallon/gallon.

```sql
UPDATE CONVERSION_SEGMENTS SET END_TIME = '2021-06-06 00:00:00' WHERE SOURCE_ID IN ( SELECT UNIT_ID FROM METERS WHERE NAME = 'Water Gallon' ) AND DESTINATION_ID IN ( SELECT ID FROM UNITS WHERE NAME = 'gallon' ) AND START_TIME = '-infinity' AND END_TIME = 'infinity';
```

- This is a reasonable test case with patterns (weeks) and slope/intercept that overlap. Here is a description:

### Units

- um1: meter unit
- ug2: first graphic unit
- ug3: second graphic unit
- ug4: third graphic unit

### Conversions

um1 -> ug2 <-> ug3 <-> ug4

### days with day segments

- d1 (1 hour segment at end)

| Hours | slope (intercept 0) |
| :---: | :---: |
| 0-7 | 2 |
| 7-23 | 3 |
| 23-24 | 5 |

- d2 (span the whole day)

| Hours | slope (intercept 0) |
| :---: | :---: |
| 0-24 | 7 |

- d3 (split day evenly)

| Hours | slope (intercept 0) |
| :---: | :---: |
| 0-12 | 13 |
| 12-24 | 17 |

### weeks

- w1

| Week | Sunday | Monday | Tuesday | Wednesday | Thursday | Friday | Saturday |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| w1 | d2 | d1 | d2 | d2 | d2 | d1 | d2 |

- w2 (all days the same)

| Week | Sunday | Monday | Tuesday | Wednesday | Thursday | Friday | Saturday |
| :-: | :-: | :-: | :-: | :-: | :-: | :-: | :-: |
| w1 | d3 | d3 | d3 | d3 | d3 | d3 | d3 |

### Overall conversions

```pre
-inf     6/3  6/4  6/6  6/7    6/10    6/16  6/17    inf
  |  10   | w1           | 20           | 30          |    um1 -> ug2
  |  40        | 50             | 60          | 70    |    ug2 <-> ug3
  |  80             | w2                      | 90    |    ug3 <-> ug4
```

When cik_vary is redone, it should show the values shown here:

### um1 -> ug2

| end of segment hour | expected value | Note |
| :---------: | :-: | :-------: |
| 6/3/2020 00 | 10 | many days |
| 6/4/2020 00 | 7 | W, d2 |
| 6/5/2020 00 | 7 | R, d2 |
| 6/5/2020 07 | 2 | F, d1 |
| 6/5/2020 23 | 3 | F, d1 |
| 6/6/2020 00 | 7 | F, d1 |
| 6/7/2020 00 | 5 | S, d2 |
| 6/16/2020 07 | 20 | |
| infinity | 30 | |

### um1 -> ug3

| end of segment hour | expected value | Note |
| :---------: | :-: | :-------: |
| 6/3/2020 00 | 400 | many days |
| 6/4/2020 00 | 280 | W, d2 |
| 6/5/2020 00 | 350 | R, d2 |
| 6/5/2020 07 | 100 | F, d1 |
| 6/5/2020 23 | 150 | F, d1 |
| 6/6/2020 00 | 250 | F, d1 |
| 6/7/2020 00 | 350 | S, d2 |
| 6/10/2020 00 | 1000 | |
| 6/16/2020 07 | 1200 | |
| 6/17/2020 23 | 1800 | |
| infinity | 2100 | |

### um1 -> ug4

| end of segment hour | expected value | Note |
| :---------: | :-: | :-------: |
| 6/3/2020 00 | 32,000 | many days |
| 6/4/2020 00 | 22,400 | W, d2 |
| 6/5/2020 00 | 28,000 | R, d2 |
| 6/5/2020 07 | 8000 | F, d1 |
| 6/5/2020 23 | 12,000 | F, d1 |
| 6/6/2020 00 | 20,000 | F, d1 |
| 6/6/2020 12 | 4,550 | d3 |
| 6/7/2020 00 | 5,950 | d3 |
| 6/7/2020 12 | 13,000 | d3, repeats until 6/9 |
| 6/8/2020 00 | 17,000 | d3, repeats until 6/10 |
| 6/10/2020 12 | 15,600 | d3, repeats until 6/16 |
| 6/11/2020 00 | 20,400 | d3, repeats until 6/17 |
| 6/16/2020 12 | 23,400 | d3 |
| 6/17/2020 00 | 30,600 | d3 |
| infinity | 189,000 | |

### SQL including seeing results

```sql
-- see data
select * from meters;
select * from units;
select * from week_patterns;
select * from day_patterns;
select * from day_segments;
select * from conversions order by  source_id, destination_id;
select * from conversion_segments order by  source_id, destination_id;
select * from cik order by source_id, destination_id;
-- Also show name of source & destination units
select (select name from units where id = source_id), (select name from units where id = destination_id), * from cik order by source_id, destination_id;
select * from cik_vary order by source_id, destination_id;
select * from cik_vary order by destination_id, start_time;
-- Also show name of source & destination units
select (select name from units where id = source_id), (select name from units where id = destination_id), * from cik_vary order by source_id, destination_id;

-- Remove data
delete from meters;
delete from conversion_segments;
delete from conversions;
delete from week_patterns;
delete from day_patterns;
delete from day_segments;
delete from cik;
delete from cik_vary;
delete from units;

-- Week 1
-- day pattern
insert into day_patterns values (DEFAULT, 'dp1', 'for ds2, ds3, ds5');
insert into day_patterns values (DEFAULT, 'dp2', 'for ds7');
-- day segments
insert into day_segments values (DEFAULT, ( SELECT id FROM day_patterns WHERE NAME = 'dp1' ), 0, 7, 2, 0, 'd2 for dp1 0-7 slope 2');
insert into day_segments values (DEFAULT, ( SELECT id FROM day_patterns WHERE NAME = 'dp1' ), 7, 23, 3, 0, 'd3 for dp1 7-23 slope 3');
insert into day_segments values (DEFAULT, ( SELECT id FROM day_patterns WHERE NAME = 'dp1' ), 23, 24, 5, 0, 'd5 for dp1 23-24 slope 5');
insert into day_segments values (DEFAULT, ( SELECT id FROM day_patterns WHERE NAME = 'dp2' ), 0, 24, 7, 0, 'd7 for dp2 0-24 slope 7');
--- week pattern
insert into week_patterns values (DEFAULT, 'w1', 'dp1: M, F; dp2: N, T, W, R, S', ( SELECT id FROM day_patterns WHERE NAME = 'dp2' ), ( SELECT id FROM day_patterns WHERE NAME = 'dp1' ), ( SELECT id FROM day_patterns WHERE NAME = 'dp2' ), ( SELECT id FROM day_patterns WHERE NAME = 'dp2' ), ( SELECT id FROM day_patterns WHERE NAME = 'dp2' ), ( SELECT id FROM day_patterns WHERE NAME = 'dp1' ), ( SELECT id FROM day_patterns WHERE NAME = 'dp2' ) );
-- units
INSERT INTO units(name, identifier, unit_represent, sec_in_rate, type_of_unit, suffix, displayable, preferred_display, note, min_val, max_val, disable_checks) VALUES ('um1', 'um1', 'quantity', 3600, 'meter', '', 'none', false, 'um1 meter unit', -999999999, 999999999, 'reject_all');
INSERT INTO units(name, identifier, unit_represent, sec_in_rate, type_of_unit, suffix, displayable, preferred_display, note, min_val, max_val, disable_checks) VALUES ('ug2', 'ug2', 'quantity', 3600, 'unit', '', 'all', false, 'ug2 graphic unit', -999999999, 999999999, 'reject_all');
INSERT INTO units(name, identifier, unit_represent, sec_in_rate, type_of_unit, suffix, displayable, preferred_display, note, min_val, max_val, disable_checks) VALUES ('ug3', 'ug3', 'quantity', 3600, 'unit', '', 'all', false, 'ug3 graphic unit', -999999999, 999999999, 'reject_all');
-- conversions
INSERT INTO CONVERSIONS VALUES ( ( SELECT id FROM units WHERE NAME = 'um1' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug2' ), false, 'um1 -> ug2' );
INSERT INTO CONVERSIONS VALUES ( ( SELECT id FROM units WHERE NAME = 'ug2' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug3' ), true, 'ug2 <-> ug3' );
-- conversion segments
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT id FROM units WHERE NAME = 'um1' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug2' ), NULL, 10, 0, '-infinity', '2020-06-03', 'segment 1 slope 10 um1 -> ug2' );
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT id FROM units WHERE NAME = 'um1' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug2' ), (select id from week_patterns where name = 'w1'), 0, 0, '2020-06-03', '2020-06-07', 'segment 2 week 1 um1 <-> ug2' );
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT id FROM units WHERE NAME = 'um1' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug2' ), null, 20, 0, '2020-06-07', '2020-06-16', 'segment 3 slope 20 um1 -> ug2' );
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT id FROM units WHERE NAME = 'um1' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug2' ), null, 30, 0, '2020-06-16', 'infinity', 'segment 4 slope 30 um1 -> ug2' );
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT id FROM units WHERE NAME = 'ug2' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug3' ), NULL, 40, 0, '-infinity', '2020-06-04', 'segment 1 slope 40 ug2 <-> ug3' );
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT id FROM units WHERE NAME = 'ug2' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug3' ), NULL, 50, 0, '2020-06-04', '2020-06-10', 'segment 2 slope 50 ug2 <-> ug3' );
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT id FROM units WHERE NAME = 'ug2' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug3' ), null, 60, 0, '2020-06-10', '2020-06-17', 'segment 3 slope 60 ug2 <-> ug3' );
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT id FROM units WHERE NAME = 'ug2' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug3' ), null, 70, 0, '2020-06-17', 'infinity', 'segment 4 slope 70 ug2 <-> ug3' );

-- Week 2: need to do week 1 first
-- day pattern
insert into day_patterns values (DEFAULT, 'dp3', 'for ds13, ds17');
-- day segments
insert into day_segments values (DEFAULT, ( SELECT id FROM day_patterns WHERE NAME = 'dp3' ), 0, 12, 13, 0, 'd13 for dp3 0-12 slope 13');
insert into day_segments values (DEFAULT, ( SELECT id FROM day_patterns WHERE NAME = 'dp3' ), 12, 24, 17, 0, 'd17 for dp3 12-24 slope 17');
--- week pattern
insert into week_patterns values (DEFAULT, 'w2', 'dp3: all days', ( SELECT id FROM day_patterns WHERE NAME = 'dp3' ), ( SELECT id FROM day_patterns WHERE NAME = 'dp3' ), ( SELECT id FROM day_patterns WHERE NAME = 'dp3' ), ( SELECT id FROM day_patterns WHERE NAME = 'dp3' ), ( SELECT id FROM day_patterns WHERE NAME = 'dp3' ), ( SELECT id FROM day_patterns WHERE NAME = 'dp3' ), ( SELECT id FROM day_patterns WHERE NAME = 'dp3' ) );
-- units
INSERT INTO units(name, identifier, unit_represent, sec_in_rate, type_of_unit, suffix, displayable, preferred_display, note, min_val, max_val, disable_checks) VALUES ('ug4', 'ug4', 'quantity', 3600, 'unit', '', 'all', false, 'ug4 graphic unit', -999999999, 999999999, 'reject_all');
-- conversions
INSERT INTO CONVERSIONS VALUES ( ( SELECT id FROM units WHERE NAME = 'ug3' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug4' ), true, 'ug3 <-> ug4' );
-- conversion segments
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT id FROM units WHERE NAME = 'ug3' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug4' ), NULL, 80, 0, '-infinity', '2020-06-06', 'segment 1 slope 80 ug3 <-> ug4' );
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT id FROM units WHERE NAME = 'ug3' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug4' ), (select id from week_patterns where name = 'w2'), 0, 0, '2020-06-06', '2020-06-17', 'segment 2 week 2 ug3 <-> ug4' );
INSERT INTO CONVERSION_SEGMENTS VALUES ( ( SELECT id FROM units WHERE NAME = 'ug3' ), ( SELECT ID FROM UNITS WHERE NAME = 'ug4' ), null, 90, 0, '2020-06-17', 'infinity', 'segment 3 slope 90 ug3 <-> ug4' );
```

```sql
-- meter: Not be formally needed but then can add readings when want.
INSERT INTO meters(name, url, enabled, displayable, meter_type, default_timezone_meter, gps, identifier, note, area, cumulative, cumulative_reset, cumulative_reset_start, cumulative_reset_end, reading_gap, reading_variation, reading_duplication, time_sort, end_only_time, reading, start_timestamp, end_timestamp, previous_end, unit_id, default_graphic_unit, area_unit, reading_frequency, min_val, max_val, min_date, max_date, max_error, disable_checks) VALUES ('m1', null, false, true, 'other', DEFAULT, DEFAULT, 'm1', 'meter 1', DEFAULT, false, false, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, ( SELECT ID FROM UNITS WHERE NAME = 'um1' ), null, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT, DEFAULT);
```

### Info

- This SQL was used to do the overall conversions.
- It is okay to only do w1 without w2 or just w2 if you add in unit ug3. Obviously the results will differ.
- It was also used without any patterns by using the conversion web page to set a conversion segment to no pattern and set the slope/intercept. In this case the days & weeks are not needed but they don't cause an issue being in the DB.
- Another variant was to remove some conversion segments from a week but this makes changes the result. The first test was only one non-pattern segment, then with a pattern, then two segments, then simple cases with chaining of conversion, etc.

## Standard developer test data with time-varying

If you insert the standard test data and don't have other items in the DB that you manually added then you should get this when you look at cik_vary (``select (select name from units where id = source_id), (select name from units where id = destination_id), * from cik_vary order by source_id, destination_id;
``):

| source | destination | source_id | destination_id | start_time | end_time | slope | intercept |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | 
| Electric_Utility | BTU | 211 | 206 | -infinity | infinity | 3412.142 | 0 |
| Electric_Utility | m³ gas | 211 | 207 | -infinity | infinity | 0.09315147659999999 | 0 |
| Electric_Utility | kWh | 211 | 209 | -infinity | infinity | 1 | 0 |
| Electric_Utility | US dollar | 211 | 217 | -infinity | infinity | 0.115 | 0 |
| Electric_Utility | MJ | 211 | 222 | -infinity | infinity | 3.6 | 0 |
| Electric_Utility | 100 w bulb | 211 | 224 | -infinity | infinity | 1 | 0 |
| Electric_Utility | euro | 211 | 226 | -infinity | infinity | 0.1012 | 0 |
| Electric_Utility | kg of CO₂ | 329 | 349 | -infinity | infinity | 0.709 | 0 |
| Electric_Utility | metric ton of CO₂ | 329 | 350 | -infinity | infinity | 0.000709 | 0 |
| Natural_Gas_M3 | BTU | 215 | 206 | -infinity | infinity | 36630.03663003663 | 0 |
| Natural_Gas_M3 | m³ gas | 215 | 207 | -infinity | infinity | 1 | 0 |
| Natural_Gas_M3 | kWh | 215 | 209 | -infinity | infinity | 10.73520288136796 | 0 |
| Natural_Gas_M3 | US dollar | 215 | 217 | -infinity | infinity | 0.25 | 0 |
| Natural_Gas_M3 | MJ | 215 | 222 | -infinity | infinity | 38.64673037292465 | 0 |
| Natural_Gas_M3 | 100 w bulb | 215 | 224 | -infinity | infinity | 10.73520288136796 | 0 |
| Natural_Gas_M3 | euro | 215 | 226 | -infinity | infinity | 0.22 | 0 |
| Water_Gallon | liter | 216 | 205 | -infinity | infinity | 3.7853996378886707 | 0 |
| Water_Gallon | gallon | 216 | 212 | -infinity | infinity | 1 | 0 |
| Temperature_Fahrenheit | Fahrenheit | 219 | 208 | -infinity | infinity | 1 | 0 |
| Temperature_Fahrenheit | Celsius | 219 | 214 | -infinity | infinity | 0.5555555555555556 | -17.77777777777778 |
| Electric_kW | kW | 221 | 220 | -infinity | infinity | 1 | 0 |
| Natural_Gas_BTU | BTU | 223 | 206 | -infinity | infinity | 1 | 0 |
| Natural_Gas_BTU | m³ gas | 223 | 207 | -infinity | infinity | 2.73e-05 | 0 |
| Natural_Gas_BTU | kWh | 223 | 209 | -infinity | infinity | 0.0002930710386613453 | 0 |
| Natural_Gas_BTU | US dollar | 223 | 217 | -infinity | infinity | 2.954545454545455e-06 | 0 |
| Natural_Gas_BTU | MJ | 223 | 222 | -infinity | infinity | 0.0010550557391808431 | 0 |
| Natural_Gas_BTU | 100 w bulb | 223 | 224 | -infinity | infinity | 0.0002930710386613453 | 0 |
| Natural_Gas_BTU | euro | 223 | 226 | -infinity | infinity | 2.6e-06 | 0 |
| Natural_Gas_BTU | kg of CO₂ | 339 | 349 | -infinity | infinity | 5.29e-05 | 0 |
| Natural_Gas_BTU | metric ton of CO₂ | 339 | 350 | -infinity | infinity | 5.29e-08 | 0
| Natural_Gas_Dollar | US dollar | 225 | 217 | -infinity | infinity | 1 | 0 |
| Natural_Gas_Dollar | euro | 225 | 226 | -infinity | infinity | 0.88 | 0 |
| Trash | kg | 229 | 210 | -infinity | infinity | 1 | 0 |
| Trash | metric ton | 229 | 213 | -infinity | infinity | 0.001 | 0 |
| Trash | kg of CO₂ | 345 | 349 | -infinity | infinity | 3.24e-06 | 0 |
| Trash | metric ton of CO₂ | 345 | 350 | -infinity | infinity | 3.24e-09 | 0 |
| Water_Gallon_Per_Minute | gallon per minute | 230 | 227 | -infinity | infinity | 1 | 0 |
| Water_Gallon_Per_Minute | liter per hour | 230 | 228 | -infinity | infinity | 227.12398 | 0 |
