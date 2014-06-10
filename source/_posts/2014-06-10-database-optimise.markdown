---
layout: post
title: "database optimise"
date: 2014-06-10 16:10
comments: true
categories: tech
---
######最近公司开发的软件遇到效率上的性能瓶颈，所以写了些代码测试一下数据库的读写效率：
------------------------
##测试目的：
 测试表项设立主键与否，以及主键类型对检索效率的影响。
##测试过程：
建立三个表，列值类型都是 int id和 varchar(100) uid,表一名为not_key_table,无主键，表二名为test_id_key设置int值为主键，表三m名为test_uid_key,设立varchar(100)为主键。分别往三个表插入一万条数据，然后从第5000条数据开始三个表连续查询1000次，统计使用的时间；
##代码：
###插入数据：
          char *st = "INSERT INTO test.not_key_table(`id`, `uid`) VALUES(%d, '%s')";
          size_t st_len = strlen(st);

          char *query=new char[st_len + 1024+1]; 
          char uid[65];
          for (int i = 0; i < 10000; i++)
          {
               dcmGenerateUniqueIdentifier(uid);
               int len = sprintf_s(query, st_len + 1024+1, (const char*)st,i+1, uid);
               if (mysql_real_query(mydata_, query, len))
               {
                    assert(false);
               }
          }
###查询代码  ：
以id作为检索条件：
     char *st = "select * from no_key_table where id=%d";
     size_t st_len = strlen(st);
     char *query=new char[st_len + 1024+1]; 
     MYSQL_RES *result = NULL;
     std::vector<string> uid_list;
     for (int i = 0; i < 1000; i++)
     {
          int len = sprintf_s(query, st_len + 1024+1, st,i+5000);
          if ( mysql_query(mydata_, query))
          {
               assert(false);
          }
          result = mysql_store_result(mydata_);
          MYSQL_ROW row = NULL;
          while((row = mysql_fetch_row(result)))
          {
               uid_list.push_back(row[1]);
          }
     }
以uid作为检索条件：
     char *st2 = "select * from not_key_tablewhere uid='%s'";
     size_t st_lens = strlen(st2);
     char *querys=new char[st_lens + 1024+1]; 
     MYSQL_RES *results = NULL;
     for (int i = 0; i < 1000; i++)
     {
          int len = sprintf_s(querys, st_lens + 1024+1, st2,uid_list[i].c_str());
          if ( mysql_query(mydata_, querys))
          {}
          results = mysql_store_result(mydata_);
     }

三个表的的测试代码基本相同，表名改一下就行，测试计时使用的是断点，在关键代码的开始和结尾设置断点，当代码运行到关键代码的开始初中断后，在vs2012的监视窗口处添加一个@clk的变量，并将值设为0，然后运行代码到第二个断点中断后，监视窗口就会显示运行这段代码所花的时钟周期。
测试结果：
    test speed of retrieve data from databse with or without primary key.
    1. 10000 line data
    ** select * from table 1000 times by retrieve id;
    *** uid table without primary key
    8,230,374
    8,550,001
    **** one time 
    9467
    8528
    *** test_id_key with int number as the primary key
    184,933
    171,449
    **** one time
    1085
    1014
    ** retrivev by uid 
    *** uid table without primary key
    15,997,671
    16,019,505
    *** test_uid_key with varchar(100) as the primary key
    115,794
    79,089
    80,074
    
    |---------------+------+------+---|
    | million clock |   id |  uid |   |
    |---------------+------+------+---|
    | table uid     |  8.3 | 16.0 |   |
    |---------------+------+------+---|
    | test_uid_key  |      | 0.08 |   |
    |---------------+------+------+---|
    | test_id_key   | 0.18 |      |   |
    |---------------+------+------+---|
    
上面是当时用emacs的org模式记得笔记，记得数字都是测试所花的时间，总的来说有主键和美主键的差别是很大的，理论上来说mysql 用的是B+树，普通查询的时间复杂度是O(n),而有主键的时间复杂度是O(log2(n)),8条数据无主键要8秒，有主键就3秒，但1000条数据有主键就只要10秒，差距是很大的。  
这次测试让我意外的是用字符串作为主键时间居然比用int型做主键效率还快，按理来说字符串比较的速度应该会比整型的比较会慢很多，但这里却相反，有点纳闷，有一部分原因或许是代码本身以及计时方式产生的误差，具体原因不清楚（知道原因的一定要和我说啊！)，但以后至少可以放心的使用不长的字符串作为主键了。
