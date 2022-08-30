
# Standby Sunucularda Manuel Datafile Ekleme

   ### İsimsiz Datafile'ı Düzeltme
   
 Standby sunucularda _STANDBY_FILE_MANAGEMENT_ parametresi _AUTO_ olarak ayarlanmamış ise primary sunucuya yeni bir datafile eklenmesi durumunda yeni eklenen datafile'ın
ismi doğru olarak _convert_ edilemez ve ilgili datafile aşağıdaki örnekteki gibi görünür ve MRP0 process'i çalışmayı durdurur.
 
```
/u01/app/oracle/product/10.1.0/dbs/UNNAMED00007
```


- Standby sunucuda eşitleme işlemi durmuşsa, yani MRP0 process'i çalışmıyor ise bunun nedeni yukarda açıklanan durum olabilir. Bunu anlamak için _sqlplus_ aracı ile
 aşağıdaki sorgu çalışıtırılarak datafile'ların isimleri kontrol edilebilir. 

```sql
 select file#,name from v$datafile;
```
```
 FILE#	NAME
   
  55    /u01/app/oracle/product/10.1.0/dbs/UNNAMED00007
```
 
 
- Sorgu sonucunda görüldüğü üzere yeni eklenen datafile doğru olarak _convert_ edilememiş. Bunu düzeltmek için primary sunucudaki datafileların ID leri ile 
   standby sunucuda oluşan isimsiz datafilein ID si karşılaştırılır, karşılaştırma işlemi için yukarıdaki sorgu primary sunucuda çalıştırılır ve isimsiz 
   datafilein isminin ne olacağı öğrenilir.
   
- Ardından datafilein nereye oluşturulacağına karar verilir ve datafilein ismi düzeltilir.
 
```sql
 ALTER DATABASE CREATE DATAFILE '/u01/app/oracle/product/10.1.0/dbs/UNNAMED00007' AS '/c_4/oradata/yenidatafile.dbf';
``` 
 
 
- Eğer standby sunucu RAC ise ve ASM kullanılıyor ise datafile oluşturmaya izin vermez. Bu durumda aşağıdaki komut kullanılabilir.
 
```sql
 alter database create datafile '/u01/app/oracle/product/10.1.0/dbs/UNNAMED00007' as new;
 ```
  
- Datafilein ismi düzeltildikten sonra MRP0 process'i başlatılır.

```sql
  alter database recover managed standby database using current logfile disconnect;
```


- Ardından _dgmgrl_ aracı ile eşitlemenin başlayıp başlamadığı kontrol edilir. Eşitleme başlamazsa standby sunucuda veritabanı kapatılıp açılabilir.
  
  
   ### _STANDBY_FILE_MANAGEMENT_ Parametresini Ayarlama
   
- Aynı problemin tekrar oluşmaması için _STANDBY_FILE_MANAGEMENT_ parametresi _AUTO_ olarak ayarlanabilir.
   
```sql  
 DGMGRL> edit database sby1 set property StandbyFileManagement = AUTO;
```  
   
   ### Primary Sunucuya Yeni Disk Eklendi İse _convert_ Ayarlarını Güncelleme
   
- Primary sunucuda yeni disk eklendi ise _db_file_name_convert_ parametresini güncellemek için **standby** sunucuda _sqlplus_ aracı ile güncel _convert_ bilgisi girilir
 
```sql   
 alter system set db_file_name_convert='/u02/oradata/sby1/','/c_4/oradata/','/u02/oradata/','/c_4/oradata/','/u03/oradata/','/c_4/oradata/' scope=spfile ;
```  
   
   
- Her ne kadar _sqlplus_ ile ilgili parametre güncellense bile bazı durumlarda _dgmgrl_ aracı yeni girilen değeri okumayabiliyor. Bundan kaçınmak için güncel _convert_ 
  bilgisi _dgmgrl_ aracı ile de ayarlanır. 

```sql
 edit database sby2 set property DbFileNameConvert = '/u02/oradata/sby1/, /c_4/oradata/, /u02/oradata/, /c_4/oradata/, /u03/oradata/, /c_4/oradata/' ; 
```

- Tüm işlemler tamamlandı. İstenirse _sqlplus_ ya da _dgmgrl_ aracı ile replikasyonun durumu kontrol edilebilir.


	
	
	
   
