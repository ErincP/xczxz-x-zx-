*&---------------------------------------------------------------------*
*& Report ZMMR_MEO_SQL_TRANSFER
*&---------------------------------------------------------------------*
*&Yazan : Erinç Pırıldar
*& Tarih : 12.08.2026
*& CNo   :
*& TCode :
*& Amaç  : MEO veri çekme programları envanterinden çekilen verileri SQL'e transfer etme.
*&---------------------------------------------------------------------*
REPORT zmmr_meo_sql.

*&---------------------------------------------------------------------*
*---- VERİ TANIMLARI -----*

DATA: BEGIN OF gs_active_table,                      " Tablo listesinde dönecek structure
        tabname TYPE zmatnr_tablolar-tabname,
      END OF gs_active_table.

DATA: gt_active_tables LIKE TABLE OF gs_active_table.   "Tablo listesini tutacak geçici tablo yapısı


DATA: dref_tab  TYPE REF TO data,
      dref_line TYPE REF TO data.

DATA: BEGIN OF gs_final_package,                "döngüde tablolardan alınan verilerin toplanacağı structure
        matnr TYPE matnr,
        zzid  TYPE zpp_meo_id,
        maktx TYPE maktx,
      END OF gs_final_package.

DATA: gt_final_package LIKE TABLE OF gs_final_package.   " Final tablosu

DATA: gv_max_matnr         TYPE matnr,                 " Delta takibini yapmaya yarayacak variable'lar.
      gv_last_processed_id TYPE zpp_tbl_meo-zzid,
      gv_max_id            TYPE zpp_tbl_meo-zzid.


TYPES: BEGIN OF gs_final_package2,                "TEST AMAÇLI
         matnr TYPE matnr,
         zzid  TYPE zpp_meo_id,
         maktx TYPE maktx,
       END OF gs_final_package2.

FIELD-SYMBOLS: <gt_data>  TYPE STANDARD TABLE,        " Dinamik veri yapısı için alan sembolleri
               <gs_data>  TYPE any,
               <gv_matnr> TYPE any,
               <gv_zzid>  TYPE any.  "Delta/Takip ID'sini okumak için.

DATA lv_maktx TYPE maktx.         " Malzeme tanımı için

DATA: gv_spras TYPE spras.        " Dil için.

DATA: gv_log_id           TYPE zpp_tbl_meo-zzid,   " LOG süreci için global değişkenler
      gv_baslangic_saati  TYPE uzeit,
      gv_bitis_saati      TYPE uzeit,
      gv_baslangic_tarihi TYPE datum,
      gv_bitis_tarihi     TYPE datum,
      gv_tanim            TYPE zpp_meo_tanim,
      meo_log             TYPE zpp_tbl_meo.


*&-----------------------------------------*---- ANA YÜRÜTME BLOĞU ------------------------------------------------------*
START-OF-SELECTION.

  PERFORM set_log_time.          "Programı çalıştırmadan evvel başlangıç tarihini ve saatini al

  gv_spras = sy-langu.           " Dil.



  SELECT MAX( zzid )                      "Log tablosundan, bu raporun son çalıştırıldığı ID'yi çekiyoruz.
           FROM zpp_tbl_meo
           WHERE tanim = 'ZMMR_MEO_SQL'
           INTO @gv_last_processed_id.
  IF gv_last_processed_id  IS INITIAL.
    gv_last_processed_id = 0.
  ENDIF.




  SELECT MAX( zzid )                      " Log tablosundan, kayıt yapan son ID'yi çekiyoruz.
    FROM zpp_tbl_meo                                       "(sistemin ulaştığı son nokta)
    INTO @gv_max_id.




  SELECT tabname FROM zmatnr_tablolar                   "Aktif tabloları çekiyoruz.
                    INTO TABLE gt_active_tables
                    WHERE aktif = 'X'.
  IF gt_active_tables IS INITIAL.
    MESSAGE 'Aktif veri tablosu bulunamadı' TYPE 'I'.
    RETURN.
  ENDIF.




  IF gv_max_id > gv_last_processed_id.        " Bizden sonra yeni hareket var mı kontrolü 


    LOOP AT gt_active_tables INTO gs_active_table.   "Eğer varsa, deltayı çekmek için tabloları sırasıyla aktif döngüye alıyoruz


      CLEAR: gv_max_matnr,
             gv_last_processed_id,
             gv_max_id.
      WRITE: / 'İşlenen Tablo: ', gs_active_table-tabname.



      TRY.
          CREATE DATA dref_tab TYPE TABLE OF (gs_active_table-tabname).   "Dinamik bellek alanlarını oluşturuyoruz.
          ASSIGN dref_tab->* TO <gt_data>.

          CREATE DATA dref_line TYPE (gs_active_table-tabname).
          ASSIGN dref_line->* TO <gs_data>.

        CATCH cx_sy_create_data_error.
          WRITE: / 'HATA: Tablo yapısı dinamik olarak oluşturulamadı!'.
          CONTINUE.
      ENDTRY.
      


      SELECT matnr, zzid FROM (gs_active_table-tabname)  "Sıradaki tablodan matnr ve zzid'yi çekelim 
              INTO CORRESPONDING FIELDS OF TABLE @<gt_data>
              WHERE zzid > @lv_last_processed_id
              AND zzid <= @lv_max_id.



      DATA(lv_lines) = lines( <gt_data> ).             "Bu tablodan kaç kayıt çektik ekrana yaz. 
      WRITE: / 'İşlenecek kayıt sayısı' , lv_lines.
      
      
      
      
      
      LOOP AT <gt_data> INTO <gs_data>.   "matnr ve zzid'leri çektiğimiz dinamik tabloda gez. 



        CLEAR: lv_maktx.         
        UNASSIGN <gv_matnr>, <gv_zzid>.                                           
        ASSIGN COMPONENT 'MATNR' OF STRUCTURE <gs_data> TO <gv_matnr>.  "Matnr sütununu dinamik değişkene ata
        ASSIGN COMPONENT 'ZZID' OF STRUCTURE <gs_data> TO <gv_zzid>.    "ZZID sütununu dinamik değişkene ata



        IF <gv_matnr> IS ASSIGNED AND <gv_matnr> IS NOT INITIAL.  "Bir matnr varsa elimizde, bunu 
          SELECT SINGLE maktx
            FROM makt
            INTO lv_maktx
            WHERE matnr = <gv_matnr>
            AND spras = gv_spras. "Sistemin mevcut dili.
        ENDIF.


        gs_final_package-matnr = <gv_matnr>.                      " Açıklamayı zzid ve mantr ile geçici tabloya atıyoruz
        gs_final_package-zzid = <gv_zzid>.
        gs_final_package-maktx = lv_maktx.
        APPEND gs_final_package TO gt_final_package.

      ENDLOOP.

      WRITE: / 'Tablo başarıyla işlendi'.
    ELSE.
      WRITE: / 'Tablo boş, kayıt bulunamadı'.
    ENDIF.

    CLEAR: dref_tab, dref_line.
    UNASSIGN: <gt_data>, <gs_data>, <gv_zzid>, <gv_matnr>.
  ENDLOOP.


  PERFORM save_log.   "İşlem sonunda log at


*&-----------------------------------------*---- ANA YÜRÜTME BLOĞU ------------------------------------------------------*



*&-----------------------------------------*---- SQL SÜRECİ ------------------------------------------------------*
  IF gt_final_package IS NOT INITIAL.                     "Tekrar eden verileri temizle
    SORT gt_final_package BY zzid matnr.
    DELETE ADJACENT DUPLICATES FROM gt_final_package COMPARING zzid matnr.
  ENDIF.


  DATA loop_final TYPE gs_final_package2.                       "SQL'den önce final paketinde gezip çalışıyor mu diye test et
  LOOP AT gt_final_package INTO loop_final.
    WRITE: / 'Materyal Adı' , loop_final-matnr.
    WRITE: / 'Materyal zzidsi' , loop_final-zzid.
    WRITE: / 'Materyal açıklaması' , loop_final-maktx.
  ENDLOOP.

*&-----------------------------------------*---- SQL SÜRECİ  ------------------------------------------------------*





*&-----------------------------------------*---- LOG SÜRECİ İÇİN FORMLAR ----------------------------------------------*


FORM set_log_time.             "Log için başlangıç ve bitiş saati ve tarihi alma
  GET TIME.
  gv_baslangic_tarihi = sy-datum.
  gv_baslangic_saati = sy-uzeit.
ENDFORM.



FORM get_next_log_id CHANGING cv_zzid TYPE zpp_tbl_meo-zzid.    "Log için sıradaki ID'yi al
  DATA: lv_number TYPE char10.
  CLEAR: cv_zzid, lv_number.
  CALL FUNCTION 'NUMBER_GET_NEXT'
    EXPORTING
      nr_range_nr             = '1'
      object                  = 'ZPP_MEO_BS'
    IMPORTING
      number                  = lv_number
    EXCEPTIONS
      interval_not_found      = 1
      number_range_not_intern = 2
      object_not_found        = 3
      quantity_is_0           = 4
      quantity_is_not_1       = 5
      interval_overflow       = 6
      buffer_overflow         = 7
      OTHERS                  = 8.
  IF sy-subrc <> 0.
    MESSAGE 'Log ID için numara aralığından değer alınmadı.' TYPE 'E'.
  ENDIF.

  CALL FUNCTION 'CONVERSION_EXIT_ALPHA_INPUT'
    EXPORTING
      input  = lv_number
    IMPORTING
      output = cv_zzid.
ENDFORM.




FORM save_log.                              " lOG alıyoruz.
  CLEAR gv_log_id.

  PERFORM get_next_log_id CHANGING gv_log_id.

  GET TIME.
  gv_bitis_tarihi = sy-datum.
  gv_bitis_saati = sy-uzeit.

  CLEAR meo_log.

  meo_log-zzid = gv_log_id.
  meo_log-tanim = 'ZMMR_MEO_SQL'.
  meo_log-bas_tr = gv_baslangic_tarihi.
  meo_log-bas_saat = gv_baslangic_saati.
  meo_log-bit_tr = gv_bitis_tarihi.
  meo_log-bit_saat = gv_bitis_saati.
  meo_log-ernam = sy-uname.
  meo_log-loekz = space.

  INSERT zpp_tbl_meo FROM meo_log.

  IF sy-subrc <> 0.
    MESSAGE 'Log başlık kaydı oluşturulamadı' TYPE 'E'.
  ENDIF.

  COMMIT WORK AND WAIT.
  MESSAGE |Log Kaydı Oluşturuldu. ID : { gv_log_id }| TYPE 'S'.

ENDFORM.


*&-----------------------------------------*---- LOG SÜRECİ  ------------------------------------------------------*
