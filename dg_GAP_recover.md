
# GAP Veren Standby Sunucularda Kurtarma İşlemleri

 Standby sunucular bazen network kaynaklı sorunlar sebebi ile bazen de disk dolması ve müdahale etmekte geç kalınması durumunda GAP verebilir ve eşitleme durabilir. Bu gibi durumlarda _primary_ sunucudan incremental yedek alınarak standby sunucuda açılarak aradaki fark kapatılabilir. Bu senaryo için gerekli işlem adımları aşağıda sıralanmıştır.
 
 - İlk adım olarak _primary_ ve _standby_ sunucuların SCN numaraları bulunur.
 
 ```
 select to_char(current_scn) from v$database ;
 ```
 ```
 primary scn:  13505724966317
 standby scn:  13505616134171
 ```
 
 - Ardından _primary_ sunucudan, _standby_ sunucu scn numarası ile yedek alınır.

 ```
 RMAN>BACKUP INCREMENTAL FROM SCN 13505616134171 DATABASE FORMAT '/yedek/ForStandby_%U' tag 'FORSTANDBY';
 ```
 
 - Alınan yedek _standby_ sunucuya kopyalanabilir veya disk paylaşımı yapılabilir.

 ```
 scp /yedek/ForStandby_* standby:/yedek/
 ```
 
 - Alınan yedekler _standby_ sunucuda kataloglanır.

 ```
 RMAN> CATALOG START WITH '/yedek/ForStandby';
 ```
 
 - _Standby_ veritabanı alınan yedeklerle _recover_ edilir.

 ```
 RMAN> RECOVER DATABASE NOREDO;
 ```
 
 - _Primary_ sunucudan _controlfile_ yedeği alınır.

 ```
 RMAN> BACKUP CURRENT CONTROLFILE FOR STANDBY FORMAT '/yedek/ForStandbyCTRL.bck';
 ```
 
 - Alınan _controlfile_ yedeği _standby_ sunucuya kopyalanabilir ve ya disk paylaşımı yapılabilir.

 ```
 scp /yedek/ForStandbyCTRL.bck standby:/yedek/
 ```
 
- _Standby_ veritabanı kapatılır ve _nomount_ modda geri açılır.

 ```
 RMAN> SHUTDOWN;
 RMAN> STARTUP NOMOUNT;
 ```

 - _Standby_ veritabanında _controlfile_ dosyası yedekten _restore_ edilir.

 ```
 RMAN> RESTORE STANDBY CONTROLFILE FROM '/yedek/ForStandbyCTRL.bck';
 ```
 
- _Standby_ veritabanı kapatılır ve _mount_ modda geri açılır

 ```
 RMAN> SHUTDOWN;
 RMAN> STARTUP MOUNT;
 ```

 - _Primary_ ve _standby_ veritabanlarında dizin yapısı özdeş ise bu adım atlanabilir. Datafile'lar iki sunucuda farklı dizinlerde tutuluyor ise datafile'ları   kataloglamak gerekli, burda iki tarafta da aynı ve farklı dizinler mevcut, farklı yerdeki datafileları ayrıca gösteriyoruz, örneğin 1,2,4,9 nolu datafile'lar iki  makinede farklı lokasyonda ilk olarak kataloglama yapılır ardından SWITCH yapılır, ne kadar dbf varsa elle ayarlıyoruz.

 ```
 RMAN> CATALOG START WITH '/c_4/oradata/';
 ```
 ```
 RMAN> switch datafile 1 to copy;
 RMAN> switch datafile 2 to copy;
 RMAN> switch datafile 4 to copy;
 RMAN> switch datafile 9 to copy;
 ```

 - _Standby_ redo logları temizlenir, bunun için standby redo log grupları öğrenilir, ardından clear edilir.

 ```
 select group#,thread#,bytes/1024/1024 as mbytes,used,archived,status from v$standby_log;
 ```
 ```
 ALTER DATABASE CLEAR LOGFILE GROUP 4; 
 ALTER DATABASE CLEAR LOGFILE GROUP 5; 
 ALTER DATABASE CLEAR LOGFILE GROUP 6;
 ```
 
 - Ardından da mrp processini başlatıyoruz.

 ```
 ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT;
 ```
 
 - Tüm işlemler tamamlandı. İstenirse _sqlplus_ ya da _dgmgrl_ aracı ile replikasyonun durumu kontrol edilebilir.
 


