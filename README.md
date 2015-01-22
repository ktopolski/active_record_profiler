ActiveRecordProfiler
====================

The `active-record-profiler` gem adds a log formatter to improve the standard SQL logging and a log subscriber to provide profiler-like tracking of SQL statements generated by application code. The log formatter makes the SQL statements in your logs look like this (imagine it's all in one log line):

    Person Load (0.3ms)  SELECT  `people`.* FROM `people`  WHERE `people`.`id` = ? LIMIT 1  [["id", 1]] 
      CALLED BY 'app/controllers/day_controller.rb:282:in `find_person''


Each SQL log entry generated by ActiveRecord will have appended the filename, line number, and function name of the nearest calling application code. The information is determined by walking up the call stack until a filename within the `/app/` directory is found. If no such filename can be found, the SQL will be logged with a location of 'Non-application code'.

Additionally, the profiler will keep track of the total time spent by all SQL calls coming from each calling location, as well as the number of time that location accessed the database. Certain SQL calls not under the direct control of the application are not counted in these statistics, such as `SHOW FIELDS`, `SET NAMES`, `BEGIN`, and `COMMIT` statements which tend to skew the timing statistics and provide less useful information about slow SQL queries.

Periodically, the profiler will dump its statistics out to a file and restart all of its counters/timers. The output file is named for the time and PID from which it was written, so that multiple threads/processes can safely write their output simultaneously.

Installation
============
Add it to your Gemfile, like so:

    gem 'active-record-profiler'

Then do `bundle install`, and then add a new initializer, `config/initializers/active_record_profiler.rb`:

    ActiveRecord::Base.logger =
        ActiveRecordProfiler::Logger.new(ActiveRecord::Base.logger)
    ActiveRecordProfiler::LogSubscriber.attach_to :active_record unless Rails.env.test?

The first line adds call-site information to ActiveRecord logging, and the second line enables profiling (except in the test environment, where it would mess up your profiling data).

Configuration
=============

## stats_flush_period ##
Control the (approximate) frequency of statistics flushes (default: `1.hour`)

    ActiveRecordProfiler::Collector.stats_flush_period = 1.hour

Note that only flushed data is available for use in the rake reports (described below). If you are running a multithreaded or multiprocess server (which covers most common rails server types), your data will be incomplete until all those threads/processes/servers have flushed their data. This limitation exists primarily to avoid the overhead of coordinating/locking during the process of serving your application's web requests.

## profile_dir ##
Directory where profile data is recorded (default: `Rails.root,join('log', 'profiler_data'`)

    ActiveRecordProfiler::Collector.profile_dir = Rails.root.join('log', 'profiler_data'

## sql_ignore_pattern ##
Any SQL statements matching this pattern will not be tracked by the profiler output, though it will still appear in the enhanced SQL logging (default: `/^(SHOW FIELDS |SET SQL_AUTO_IS_NULL|SET NAMES |EXPLAIN |BEGIN|COMMIT|PRAGMA )/`)

    ActiveRecordProfiler::Collector.sql_ignore_pattern = /^SET /x


Reports
=======
To see a top-100 list of what SQL statements your application is spending its time in, run the following rake task:

    rake profiler:aggregate max_lines=100 show_sql=true

This will return a list of the SQL which is taking the most time in your application in this format:

    <file path>:<line number>:in <method name>: <total duration>, <call count>, <max single call duration>

This will aggregate all of the profiler data you have accumulated; in order to limit the timeframe of the data, use the `prefix` option to specify a partial date/time:

    rake profiler:aggregate max_lines=100 show_sql=true prefix=2010-06-20-10  # data from June 20 during the 10am hour (roughly)

Each thread running the profiler flushes its stats periodically, and there is a rake task to combine multiple profiler data files together in order to keep the number of data files down to a manageable number. A good way to manage the data files on a server is to set up a cron task to run the following command once per hour or once per day:

    rake profiler:aggregate compact=<'hour' or 'date'> RAILS_ENV=qa

Compacting by hour will result in a single file for each hour any process dumped its stats. Compacting by day will result in a single file for each day. When using the `prefix` option to generate a profiler report, you cannot specify an hour if you have compacted your data by date instead of hour (the prefix matching operates on the file names, which will not have hours if they have been compacted by date).

You can clear out all profiler data using the following command:

    rake profiler:clear_data
  
If you want programmatic access to the profiler data, check out the source code for the rake tasks in `lib/active-record-profiler/tasks.rake`.


=======
HTML Reports
============

The profiler includes some view helpers to make it easy for your application to generate a sortable HTML table of profiler information. The core helper method generates a table based on an `ActiveRecordProfiler::Collector` object. In its simplest form, it can be called from a view like this:

    <div id="#profiler">
      <%= profiler_report(params) %>
    </div>
  
You can also set up a bunch of options by passing a set of configuration keys:

    profiler_report(params, options)
  
The available options include:

Key | Description
--- | -----------
:date | year, year-month, year-month-day, year-month-day-hour used to filter the profiler data; defaults to `Today` (String)
:sort | ActiveRecordProfiler::(`DURATION`&#124;`COUNT`&#124;`LONGEST`&#124;`AVG_DURATION`) specifying which field to sort the report by; defaults to `DURATION` (Constant/Integer)
:max_rows | Maximum number of table rows to output; in other words, report on the top max_rows SQL statements; defaults to 100 (Integer)
:collector | object representing the profile data to use in building the report; defaults to an empty collector using the configured profile data directory (`ActiveRecordProfiler::Collector`)
:table | css class applied to the report &lt;table&gt; element; defaults to `nil`
:header_row | css class applied to the report's header row; defaults to `nil`
:row | css class applied to the report's data rows; defaults to `nil`
:link_locations | `true`/`false` value indicating whether to build textmate links to the source code whence a given piece of SQL came; defaults to false
  
An easy way to support filtering of report data by month/date/hour is to use a view like this:

    <%= profiler_date_filter_form(params) %>
    <%= profiler_report(params) %>
  
And if you use TextMate, then you may want to throw in some extra goodies to generate links to the actual source code files and lines where the SQL was triggered (Note: the current javascript requires jQuery):

    <%= profiler_date_filter_form(params) %>
    <%= profiler_report_local_path_form %>
    <%= profile_report_local_path_javascript %>
    <%= profiler_report(params, {:link_locations => true}) %>


Miscellaneous
=============

Copyright (c) 2010 Gist, Inc.
Copyright (c) 2015 Benjamin Turner
