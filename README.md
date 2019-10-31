# utl-PASSTHRU-to-mysql-and-select-rows-based-on-a-SAS-dataset-without-loading-the-SAS-daatset-into-my
PASSTHRU to mysql and select rows based on a SAS dataset without loading the SAS daatset into mySQL
    SAS Forum: PASSTHRU to mysql and select rows based on a SAS dataset without loading the SAS daatset into mySQL;

    Note the maximum length of a macro string is 64k bytes which is two short for 100,000 values a single query

    I create multiple macro strings and call mySQL for each macro string

    We use passthru to mysql for maximum performance, for generality I did not use the SQL between operation.

    This is an attempt to join a SAS table to a mySQL table without

             1. Exporting a mySQL table to SAS and doing the join in SAS
             2. Loaing the SAS table into mySQL and doing the join in mySQL

    This should be especially fast when SAS integrates DOSUBL with the datastep. Currently
    SAS has a very poor implementation of DOSUBL.

    github
    https://tinyurl.com/yxf5rh9c
    https://github.com/rogerjdeangelis/utl-PASSTHRU-to-mysql-and-select-rows-based-on-a-SAS-dataset-without-loading-the-SAS-daatset-into

    SAS Forum  (related to)
    https://tinyurl.com/y34c4frk
    https://communities.sas.com/t5/SAS-Programming/Pulling-Data-from-SQL-Server-Based-on-List-of-Values-in-SAS/m-p/600440


    *_                   _
    (_)_ __  _ __  _   _| |_ ___
    | | '_ \| '_ \| | | | __/ __|
    | | | | | |_) | |_| | |_\__ \
    |_|_| |_| .__/ \__,_|\__|___/
            |_|
    ;


    data have ;
      input id @@;
      cards4 ;
    10 20 30 40 50 60 70 80 90 100
    ;;;;
    run ;

    WORK.HAVE total obs=10

       ID

       10
       20
       30
       40
       50
       60
       70
       80
       90
      100


    libname mysqllib mysql user=root password="xxxxxxxx" database=sakila port=3306;

    * in case it exists;
    proc sql;
      drop table mysqllib.oneHun;
    ;quit;

    data mysqllib.oneHun;
      do id=1 to 100;
        name=cats('nam',put(id,z3.));
        output;
      end;
      stop;
    run;quit;

    libname mysqllib clear;


    Mysql Table onehun

                   | RULE
     ID     NAME   |
                   |
      1    nam001  |
      2    nam002  |
      3    nam003  |
      4    nam004  |
      5    nam005  |
      6    nam006  |
      7    nam007  |
      8    nam008  |
      9    nam009  |

     10    nam010  |  Match with SAS have

     11    nam011  |
     12    nam012  |
     13    nam013  |
     14    nam014  |
     15    nam015  |
     16    nam016  |
     17    nam017  |
     18    nam018  |
     19    nam019  |

     20    nam020  |  Match with SAS have
     ...

    100   nam0100  |  Match with SAS have


    *            _               _
      ___  _   _| |_ _ __  _   _| |_
     / _ \| | | | __| '_ \| | | | __|
    | (_) | |_| | |_| |_) | |_| | |_
     \___/ \__,_|\__| .__/ \__,_|\__|
                    |_|
    ;

    WORK.BASE total obs=10

       ID     NAME

       10    nam010
       20    nam020
       30    nam030
       40    nam040
       50    nam050
       60    nam060
       70    nam070
       80    nam080
       90    nam090
      100    nam100

    *
     _ __  _ __ ___   ___ ___  ___ ___
    | '_ \| '__/ _ \ / __/ _ \/ __/ __|
    | |_) | | | (_) | (_|  __/\__ \__ \
    | .__/|_|  \___/ \___\___||___/___/
    |_|
    ;

    /* maybe add "if mod(_n_,5)=0 or end=dne" for last partial list not tested */

    data _null_;

      if _n_=0 then do;%let rc=%sysfunc(dosubl('
         proc datasets lib=work nolist;
              delete base;
         run;quit;
         '));
      end;

      length lst $200;
      retain grp 0 lst;

      set have end=dne;

      if mod(_n_-1,5)<=4 then do; lst=catx(',',lst,id); end;

      if mod(_n_,5)=0 then do;
          call symputx('lst',lst);
          rc=dosubl('
             proc sql;
               connect to mysql (user=root password="xxxxxxxxxx"
                    database=sakila);
               create table addnum as select * from connection to mysql(
                select * from  oneHun where id in (&lst)
               ) ;
               disconnect from mysql
            ;quit;
            proc append data=addNum base=base;
            run;quit;
          ');

          lst="";

      end;

    run;quit;

    *_
    | | ___   __ _
    | |/ _ \ / _` |
    | | (_) | (_| |
    |_|\___/ \__, |
             |___/
    ;

    SYMBOLGEN:  Macro variable LST resolves to 10,20,30,40,50
    NOTE: Table WORK.ADDNUM created, with 5 rows and 2 columns.

    NOTE: PROCEDURE SQL used (Total process time):
          real time           0.01 seconds


    NOTE: Appending WORK.ADDNUM to WORK.BASE.
    NOTE: BASE data set does not exist. DATA file is being copied to BASE file.
    NOTE: There were 5 observations read from the data set WORK.ADDNUM.
    NOTE: The data set WORK.BASE has 5 observations and 2 variables.
    NOTE: PROCEDURE APPEND used (Total process time):
          real time           0.01 seconds


    SYMBOLGEN:  Macro variable LST resolves to 60,70,80,90,100
    NOTE: Table WORK.ADDNUM created, with 5 rows and 2 columns.

    NOTE: PROCEDURE SQL used (Total process time):
          real time           0.01 seconds


    NOTE: Appending WORK.ADDNUM to WORK.BASE.
    NOTE: There were 5 observations read from the data set WORK.ADDNUM.
    NOTE: 5 observations added.
    NOTE: The data set WORK.BASE has 10 observations and 2 variables.
    NOTE: PROCEDURE APPEND used (Total process time):
          real time           0.00 seconds


    NOTE: There were 10 observations read from the data set WORK.HAVE.
    NOTE: DATA statement used (Total process time):
          real time           0.63 seconds
          user cpu time       0.20 seconds

    742 !     quit;

