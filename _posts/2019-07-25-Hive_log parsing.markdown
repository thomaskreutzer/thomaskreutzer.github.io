---
layout: post
title:  "Hive log parsing"
date:   2019-07-25 08:27:29 -0400
categories: hdp hive
author: Thomas Kreutzer
---
### Description:
I work with clients that have not installed SmartSense and we need to upload logs for support cases. Data must be removed such as table names, IP addresses, kerberos realms, user names, database names. 


This is a quick script, it would likely require tweaking to your requirements. This is by no means the most optimal method, but will get you by in a pinch and removes SQL queries from the log even if on multiple lines. 

{% highlight python %}
'''
HIVE LOG PARSER
Parses out the hiveserver log files including multi-line SQL.
May require some tweaks, please check the output of your log files.
'''
import re
import os

file_search='hive'
write_to_file = True
skip_next_expression = False

def coll_files(file_nm_contains):
    myarr = []
    for r, d, f in os.walk("."):
        for file in f:
            if file_nm_contains in file:
                myarr.append(file)
    return myarr

def parse_line(line):
    p = re.compile('10.30.(\d+).(\d+)')
    newline = p.sub('IP', line)
   
    p2 = re.compile('x(\d{1,7})')
    newline = p2.sub('usr', newline)
   
    p3 = re.compile('xvic(\d{1,7})')
    newline = p3.sub('usr', newline)
   
    p4 = re.compile('hostserverprefix')
    newline = p4.sub('svr', newline)
   
    p5 = re.compile('.some.domain')
    newline = p5.sub('', newline)
   
    p6 = re.compile('SOME.REALM.YOURENV.COM')
    newline = p6.sub('REALM.COM', newline)
   
    p7 = re.compile('specific_id,')
    newline = p7.sub('user', newline)
   
    p8 = re.compile('db=(.*)\s{0,2}pat=')
    newline = p8.sub('db=sdb pat=*', newline)
   
    p8 = re.compile('ugi=(.*)\sip=')
    newline = p8.sub('ugi=id sip=', newline)
   
    p9 = re.compile('tbl=(.*)')
    newline = p9.sub('tbl=tbl', newline)
   
    p10 = re.compile('owner=(.*)\s{1}')
    newline = p10.sub('owner=usr ', newline)
   
    p11 = re.compile('get_function:\s*(.*\..*)\n')
    newline = p11.sub('get_function:rmfunct\n', newline)
   
    p12 = re.compile('get_database:\s*(.*)\n')
    newline = p12.sub('get_database: db\n', newline)
   
    p13 = re.compile('db=(.*)\s*tbl=')
    newline = p13.sub('db=db tbl=', newline)
   
    p14 = re.compile('Parsing command: (.*)\n')
    newline = p14.sub('Parsing command: rmvd\n', newline)
   
    p15 = re.compile('hdfs:\/\/.*\n')
    newline = p15.sub('hdfs://\n', newline)
   
    p16 = re.compile('ALTER TABLE(.*)\n')
    newline = p16.sub('ALTER TABLE removed \n', newline)
   
    p17 = re.compile('(dbname:.*?),')
    newline = p17.sub('dbname:db,', newline)
   
    p18 = re.compile('(tablename:.*?),')
    newline = p18.sub('tablename:tbl,', newline)
   
    p19 = re.compile('INSERT INTO TABLE(.*)\n')
    newline = p19.sub('INSERT INTO TABLE rmvd\n', newline)
   
    p20 = re.compile('SHOW TABLES(.*)\n')
    newline = p20.sub('SHOW TABLES rmvd\n', newline)
   
    p21 = re.compile('DDLTask: got data for(.*)\n')
    newline = p21.sub('DDLTask: got data for rmvd\n', newline)
   
    p22 = re.compile('DDLTask: written data for(.*)\n')
    newline = p22.sub('DDLTask: written data for rmvd\n', newline)
   
    p23 = re.compile('create_table_core(.*)\n')
    newline = p23.sub('create_table_core rmvd\n', newline)
   
    p24 = re.compile('(\sTable\s*.*)\s*not found:\s*(.*).(.*)table not found\n')
    newline = p24.sub('Table tbl not found db.tbl table not found\n', newline)
   
    p25 = re.compile('CREATE(.*)\n')
    newline = p25.sub('CREATE removed\n', newline)
   
    return newline
 
for myfile in coll_files(file_search):
    f = open(myfile)
    fout = open("parsed." + myfile, 'w')
    line = f.readline()
    while line:
       matchObj = re.match(r'(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2},\d{3}\s*INFO.*)CREATE', line)
       if matchObj:
           write_to_file=False
           skip_next_expression = True
           '''
       else:
            print "No match!!"'''
       line = f.readline()
      
       if skip_next_expression != True:
           #print "do not skip
           ''' Here we will actually start checking for the end of the SQL statement after the first line was skipped. '''
           matchObj2 = re.match(r'(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2},\d{3}\s*INFO.*)', line)
           if matchObj2:
               write_to_file = True
       else:
           skip_next_expression = False
           #print "skip"
          
       if write_to_file:
           fout.write( parse_line(line) )
    f.close()
{% endhighlight %}