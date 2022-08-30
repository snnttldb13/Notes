
# Oracle RAC Veritabanlarında Datafile Taşıma İşlemleri


 ## Primary RAC sunucularda datafile taşıma 
 
- Taşınacak datafile lar seçilir (OMF olmayan datafaliler ile işlem yapılmaz).

```
 örneğin 141 nolu datafile'ı taşımak için 
```


- Taşınacak olan ilk datafile offline edilir.

```sql
 alter database datafile 141 offline;
```


- Ardından RMAN ile taşınmak isteden datafile yeni disk grubuna kopyalanır.

```
RMAN> copy datafile 141 to '+DATA2';
```


- Taşıma sonrasında işlem yapılan datafile'ın adı _sqlplus_ aracı kullanılarak değiştirilir. Bir önceki adımda RMAN komutunun çıktısında girdi olan datafile ve 
  çıktı olan datafile'ların isimleri veriliyor, ordan gelen isiler ile: 

```sql
 ALTER DATABASE RENAME FILE 'input datafile' TO 'output datafile';
```

  Örneğin:

```sql
 ALTER DATABASE RENAME FILE '+FRA/rac/datafile/krmdaudit.27346.1044110615' TO '+DATA2/rac/datafile/krmdaudit.272.1076962841';
```


- Bir önceki adımda yapılan RENAME FILE işlemi eski lokasyonda bulunan datafile'ı diskten siler. Bunun için ayrıca işlem yapmaya
 gerek yoktur. Alert log incelenirse silme işleminin yapıldığı görülebilir.


- Taşınan datafile _sqlplus_ aracı kullanılarak recover edilir, bir önceki adımda verilen yeni isim/yol kullanılır.

```sql
 RECOVER DATAFILE '+DATA2/rac/datafile/krmdaudit.272.1076962841';
```	


- Datafile online edilir.

```sql
 ALTER DATABASE DATAFILE '+DATA2/rac/datafile/krmdaudit.272.1076962841' ONLINE;
```



 ## Standby RAC sunucularda datafile taşıma


- Standby sunucularda datafile taşımak için datafile'i offline etmeye gerek yoktur. Ancak _standby_file_management_ parametresini _manual_ olarak değiştirmek gereklidir.
 Bunun için _sqlplus_ aracı kullanılır.
 
```sql
 alter system set standby_file_management=manual;
```


- Taşınacak datafile lar seçilir (OMF olmayan datafaliler ile işlem yapılmaz).

```
örneğin 99 nolu dbf 
```

- RMAN ile taşınmak istenen datafile yeni disk grubuna kopyalanır.

```sql
RMAN> copy datafile 99 to '+DATA2';
```


- Taşıma sonrasında işlem yapılan datafile'ın adı _sqlplus_ aracı kullanılarak değiştirilir. Bir önceki adımda RMAN komutunun çıktısında girdi olan datafile ve 
  çıktı olan datafile'ların isimleri veriliyor, ordan gelen isiler ile: 

```sql
 ALTER DATABASE RENAME FILE 'input datafile' TO 'output datafile';
```

  Örneğin:

```sql
 ALTER DATABASE RENAME FILE '+FRA/rac/datafile/krmdlob.19532.1059850777' TO '+DATA2/rac/datafile/krmdlob.216.1436967411';
```


- Bir önceki adımda yapılan RENAME FILE işlemi eski lokasyonda bulunan datafile'ı diskten siler. Bunun için ayrıca işlem yapmaya
 gerek yoktur. Alert log incelenirse silme işleminin yapıldığı görülebilir.


- Taşıma işlemlerinin ardından işlemlere başlarken _manuel_ olarak değiştirdiğimiz _standby_file_management_ parametresini eski haline alıyoruz.

```sql
 alter system set standby_file_management=AUTO ; 
```

