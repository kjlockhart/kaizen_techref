ctTrends
========

.. important::
  ctTrends is heart of the Kaizen.  It must support fast query and update operations. 
  All accessing code shall avoid the following situations which degrade performance:

  * unindexed queries.  The _id has been carefully crafted to answer all common queries through
    clever regex requests.
  * asking for part of a day bucket.  Instead retrieve the entire bucket and discard any unwanted 
    samples client-side.
  * using ctTrends where ctSummary would be sufficient.
  
.. tip::
  Use *upsert* and upsert operators ($inc, $set, ...) to avoid read/write round-trips of the data, 
  and the associated document-locking + read-write race conditions.  These operators provide 
  atomic document updates. 


Latest
------

Trend logs are stored, not as individual samples, but as sets of date-related samples - called 
*day buckets*.  This format mirrors how the data is used (i.e. by day, or range of days), and  
greatly reduces storage space, by factoring out the significant overhead needed to 
identify individual samples.  Samples are time-value pairs.  Sample values may be any BSON 
datatype (usually INT, FLOAT, BOOLAN. STRING[future]).  Day Buckets typically contain 24, 96, 
288, or 1440 samples representing hourly, 15 min, 5 min, and 1 min sample polling frequencies.  
Unused timeslots are obmitted.  Both Polled and COV trends are easily accommodated.

.. sidebar:: ctTrends 

  | "_id" : "2156.100.TL10-2015-08-12", 
  | "l" : NumberInt(2156), 
  | "d" : NumberInt(100), 
  | "t" : "TL", 
  | "i" : NumberInt(10), 
  | "dt" : "2015-08-12", 
  | "rev" : "5", 
  | "ts" : ISODate("2015-09-09T00:02:18.792-0800"), 
  | "00:00:00" : 100.0, 
  | "01:00:00" : 100.0, 
  | "02:00:00" : 100.0, 
  |   ...
  | "23:00:00" : 100.0

_id (primary key)
  A composite of the <building>.<device>.TL<instance>-<date> which are 
  also repeated as individual fields for programmer convenience as follows.
  
l (location_id):
  [alias building_id].  Kaizen unique building_id.  Valid range is 1..N
  
d (device_id)
  BAS unique controller_id.  Originates from BACnet.  Valid range is 1..N when data comes from a 
  controller.  0 is used for internally generated data (like CTLs) as it cannot conflict with any 
  real controller.

t (memonic for trend log)
  Originates from BACnet.  Retained for object refence compatibily with ctObjects.  Other common 
  memonics (found in ctObjects) are DEV (Device), MT (Multi-trend), and AI/AO (Analog Input, 
  Analog Output), and BI/BO (Binary Input, Binary Output). 

i (instance)
  Controller unique trend_id.  Originates from BACnet.  Valid range is 1..N
  
dt (date)
  The date (in YYYY-MM-DD format) associated with the accompanying samples.  Pulling the date out 
  of the sample timestamp saves space and makes the sample time human-friendly.
  
rev (revision)
  Internal revision scheme identifying the format of the day bucket.  Rarely (if ever) used. 
  Its importance faded as the design of the day bucket stablized on the current format:
  * '5' - standard bucket as generated by intake_trends.
  * '5C'- [deprecated] Indicated a calculated trend log (CTL) 
  * 'IMP'- [deprecated] Indicated an Imported trend log

ts (timestamp) 
  The server time marking when the data was processed by the Intake Server.
  
HH:MM:SS (sample time)
  The time portion of the samples' timestamp (in building controller local time). It marks  
    when the sample was taken by the controller.  
    
.. note:: Sample times are always in Building Local Time (BTL), not server time, not UTC time.  This is 
  due to the complexities associated with data that traverses multiple timezones.  A number of time tracking 
  approaches were tried before abandoning them all for the sweet-simplity of Building Local Time.
  This is the time the controller says it is - and the only time the user really cares about.
  The controller is deemed correct.  Therefore the sample was taken at the time the controller says 
  it was taken.  [full stop.]  



Summary
-------

.. sidebar:: ctSummary

  | "_id" : "30.230.TL56", 
  | "name" : "T3_CHIL_MODE_MT"
  | "first" : "2013-05-02"
  | "last" : "2016-02-15", 
  | "ts" : ISODate("2016-02-15T21:47:22.428-0800"), 

ctTrends is a massive collection (100's of millions of day buckets).  Queries to find the first 
or last day bucket of a trend log is very expensive.  ctSummary exists to provide a cheap means 
to obtain this often needed information.  It is maintained via a cooperation between the 
Intake Servers - intake_trends and intake_objects, and the background (cron) tasks - 
intake/mt_summary.py and ct_stats/tl_storage_lite.py.  Individually none of them have enough 
information, but collectively they keep ctSummary up-to-date in real-time and prevent items 
*falling through the cracks* by rebuilding it completely each week.
  
_id (primary key) 
  A composite of the <building>.<device>.TL<instance> as used elsewhere.

name 
  Name of the trend log as obtained from ctObjects::Latest by intake_objects.py as the TL & MT objects 
  arrive in the intake queue.  Used by Vault (list of TLs) and TLViewer.
  
first
  Earliest dated day bucket.  Determines the oldest data that is available in the trend log.
  
last
  Latest dated day bucket.  Determines the newest data that is available in the trend log.
  Together *first* and *last* define the range of time covered by the trend, and thus the dates 
  that can be queried from the trend.
  
ts (timestamp) 
  The server time marking when this document was last updated.
  

Write Concern
-------------

MongoDB has a configurable write concern, to allow balancing the importance of 
guaranteeing that all writes are fully recorded in the database against the speed of the insert. 
For example, if you issue writes without ack-required, the write operations will return very fast, 
but you cannot be certain the write succeeded. Conversely, a write with ack_required, 
will not return as quickly but you can be certain the write succeeded.

We use w=1 to to ensure that MongoDB acknowledges inserts. We choose not to wait for acknowledgement 
of the write to journal, or the write to propograte to a majority of replica set members. 
