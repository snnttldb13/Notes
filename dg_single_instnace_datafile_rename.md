
# Single Instance Standby Veritabanında Datafile Taşıma

  Veritabanı sunucusunda mevcut durumda kullanılan disk dolmuş ve yeni bir eklenmiş olabilir. Bu durumda mevcut bazı datafile'ları taşıyarak diskte boş yer açmak gerekebilir.
Bunun için gerekli işlemler ve sonrasında yeni diskin otomatik olarak kullanılabilmesi için gerekli ayarlar aşağıda anlatılmıştır.

- MRP0 process'i eğer çalışıyor ise durdurulur. Kontrol etmek için _sqlplus_ aracı ile aşağıdaki sorgu çalıştırılır.

```
select process,status from v$managed_standby;
```
```
PROCESS   STATUS
--------- ------------
ARCH      CLOSING
ARCH      CONNECTED
ARCH      CLOSING
ARCH      CLOSING
RFS       IDLE
MRP0      APPLYING_LOG
RFS       IDLE

7 rows selected.
```

- MRP0 process'i eğer çalışıyor ise durdurmak için _sqlplus_ aracı ile aşağıdaki gibi durdurulur.

```
 alter database recover managed standby database cancel;
```

- Ardından _standby_file_management_ parametresi _manuel_ olarak ayarlanır.

```
 alter system set standby_file_management=manual ;
```

- Parametre değişikliğinin ardından taşınmak istenen datafile'lar işletim sistemi komutları ile yeni konumlarına taşınırlar.

```
 mv /c_4/oradata/KRMDAUDIT23.DBF /u02/oradata/
```

- Taşıma bittikten sonra taşınan datafile'ların isimleri  _sqlplus_ aracı ile güncellenir.

```
 ALTER DATABASE RENAME FILE '/c_4/oradata/KRMDAUDIT23.DBF' TO '/u02/oradata/KRMDAUDIT23.DBF';
```

- İsim değişikliğinin ardından yeni eklenecek datafileların da otomatik olarak yeni diske eklenmesi için gerekli parametre değişikliklerini yapmak gerekli. İlk olarak _sqlplus_ aracı ile _db_file_name_convert_ 
 parameterisini güncelliyoruz.
 
```
 alter system set db_file_name_convert='/c_4/oradata/orcl/','/u02/oradata/','/c_4/oradata/','/u02/oradata/' scope=spfile ;
```

- Ardından _dgmgrl_ aracı ile _DbFileNameConvert_ property'sini güncelliyoruz.

```
 edit database sby1 set property DbFileNameConvert = '/c_4/oradata/orcl/, /u02/oradata/, /c_4/oradata/, /u02/oradata/' ; 
```

- Parametre güncellemeleri tamamlandı. Şimdi de işlemlere başlarken _manuel_ olarak değiştirdiğimiz _standby_file_management_ parametresini eski haline alıyoruz.

```
 alter system set standby_file_management=AUTO ; 
```

- Parametre değişiklikleri yaparken _scope=spfile_ olarak kullandığımız için veritabanını yeniden başlatmamız gerekli. 

```
 shu immediate ;
 startup mount;
```

- Veritabanı kapanıp tekrar mount modda başladıktan sonra MRP0 process'i de otomatik olarak başlamış olmalı, kontrol edebiliriz.



