ActiveRecordProfiler
====================

ActiveRecordProfiler monkey-patches 
ActiveRecord::ConnectionAdapters::AbstractAdapter both to improve the 
standard SQL logging and to provide profiler-like tracking of SQL
statements generated by application code.

Each SQL log entry generated by ActiveRecord will have appended the 
filename, line number, and function name of the nearest calling 
application code. The information is determined by walking up the call
stack until a filename within the /app/ directory is found. If no such
filename can be found, the SQL will be logged with a location of 
'Non-application code'.

Additionally, the profiler will keep track of the total time spent by all
SQL calls coming from each calling location, as well as the number of time
that location accessed the database. Certain SQL calls not under the
direct control of the application are not counted in these statistics,
such as "SHOW FIELDS", "SET NAMES", "BEGIN", and "COMMIT" statements which
tend to skew the timing statistics and provide less useful information
about slow SQL queries.

Periodically, the profiler will dump its statistics out to a file and
restart all of its counters/timers. The output file is named for the time
and PID from which it was written, so that multiple processes can safely
write their output simultaneously.

Installation
============
Add it to your Gemfile, do `bundle install`, and add the following to any `config/environments/*.rb*` files where you want to enable the profiler (or add it to `config/application.rb` if you want it enabled everywhere) :

    config.log_formatter = ActiveRecordProfiler::LogFormatter.new

Then add a new initializer, `config/initializers/active_record_profiler.rb`:

    ActiveRecord::Base.logger =
        ActiveRecordProfiler::Logger.new(ActiveRecord::Base.logger)
    ActiveRecordProfiler::LogSubscriber.attach_to :active_record


The first line adds call-site information to ActiveRecord logging, and the second line enables profiling.

Configuration
=============
Control the (approximate) frequency of statistics flushes (default: `1.hour`)

    ActiveRecordProfiler::Collector.stats_flush_period = 1.hour

Directory where profile data is recorded (default: `Rails.root,join('log', 'profiler_data'`)

    ActiveRecordProfiler::Collector.profile_dir = Rails.root,join('log', 'profiler_data'

Any SQL statements matching this pattern will not be tracked by the 
profiler output, though it will still appear in the enhanced SQL logging
(default: `/^(SHOW FIELDS |SET SQL_AUTO_IS_NULL|SET NAMES |EXPLAIN |BEGIN|COMMIT)/`)

    ActiveRecordProfiler::Collector.sql_ignore_pattern = /^SET /x

If you don't want to use the JSON gem to store your profiler data, you can
use the FasterCSV gem instead, but due to field length constraints in 
FasterCSV's parsing code, some of your SQL may be truncated.

    ActiveRecordProfiler::Collector.storage_backend = :fastercsv


Reports
=======
To see a top-100 list of what SQL statements your application is spending its
time in, run the following rake task:

    rake profiler:aggregate RAILS_ENV=qa max_lines=100 show_sql=true

This will return a list of the SQL which is taking the most time in your 
application in this format:

    <file path>:<line number>:in <method name>: <total duration>, <call count>, <max single call duration>

This will aggregate all of the profiler data you have accumulated; in order 
to limit the timeframe of the data, use the `prefix` option to specify a
partial date/time:

    rake profiler:aggregate RAILS_ENV=qa max_lines=100 show_sql=true prefix=2010-06-20-10  # data from June 20 during the 10am hour (roughly)

Each process running the profiler flushes its stats periodically, and there
is a rake task to combine multiple profiler data files together in order to 
keep the number of data files down to a manageable number. A good way to 
manage the data files on a server is to set up a cron task to run the 
following command once per hour or once per day:

    rake profiler:aggregate compact=<'hour' or 'date'> RAILS_ENV=qa

Compacting by hour will result in a single file for each hour any process 
dumped its stats. Compacting by day will result in a single file for each 
day. When using the `prefix` option to generate a profiler report, you
cannot specify an hour if you have compacted your data by date instead of
hour (the prefix matching operates on the file names, which will not have
hours if they have been compacted by date).

You can clear out all profiler data using the following command:

    rake profiler:clear_data RAILS_ENV=qa
  
If you want programmatic access to the profiler data, check out the source
code for the rake tasks in `lib/active-record-profiler/tasks.rake`.


HTML Reports
============

The profiler includes some view helpers to make it easy for your application
to generate a sortable HTML table of profiler information. The core helper
method generates a table based on an ActiveRecordProfiler::Collector object.
In its simplest form, it can be called from a view like this:

  <div id="#profiler">
    <%= profiler_report(params) %>
  </div>
  
The full set of parameters available looks like this:

  profiler_report(page_parameters, date_prefix, sort_id, max_rows, profiler_collector, css_options)
  
parameters:
  page_parameters: this is generally the request parameters, used to build the report-sorting links (HashWithIndifferentAccess)
  options: hash containing optional settings for the report; supported keys:
  
    :date : year, year-month, year-month-day, year-month-day-hour used to filter the profiler data; defaults to Today (String)
    :sort : ActiveRecordProfiler::(DURATION|COUNT|LONGEST|AVG_DURATION) specifying which field to sort the report by; defaults to DURATION (Constant/Integer)
    :max_rows : Maximum number of table rows to output; in other words, report on the top max_rows SQL statements; defaults to 100 (Integer)
    :collector : object representing the profile data to use in building the report; defaults to an empty collector using the configured profile data directory (ActiveRecordProfiler::Collector )
    :table : css class applied to the report <table> element; defaults to nil
    :header_row : css class applied to the report's header row; defaults to nil
    :row : css class applied to the report's data rows; defaults to nil
    :link_locations : true/false value indicating whether to build textmate links to the source code whence a given piece of SQL came; defaults to false
  
An easy way to support filtering of report data by month/date/hour is to 
use a view like this:

  <%= profiler_date_filter_form(params[:date], params[:sort]) %>
  <%= profiler_report(params) %>
  
And if you use TextMate, then you may want to throw in some extra goodies
to generate links to the actual source code files and lines where the SQL
was triggered (Note: the current javascript requires jQuery):

  <%= profiler_date_filter_form(params[:date], params[:sort]) %>
  <%= profiler_report_local_path_form %>
  <%= profile_report_local_path_javascript %>
  <%= profiler_report(params, {:link_locations => true}) %>


Miscellaneous
=============

Copyright (c) 2010 Gist, Inc.
Copyright (c) 2015 Benjamin Turner
