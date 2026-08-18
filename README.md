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
DATA: lv_max_matnr         TYPE matnr,
      lv_last_processed_id TYPE zpp_tbl_meo-zzid,
      lv_max_id            TYPE zpp_tbl_meo-zzid.
TYPES: BEGIN OF gs_final_package2,                "döngüde tablolardan alınan verilerin toplanacağı structure
         matnr TYPE matnr,
         zzid  TYPE zpp_meo_id,
         maktx TYPE maktx,
       END OF gs_final_package2.

DATA: gt_final_package LIKE TABLE OF gs_final_package.
FIELD-SYMBOLS: <gt_data>  TYPE STANDARD TABLE,
               <gs_data>  TYPE any,
               <gv_matnr> TYPE any,
               <gv_zzid>  TYPE any.  "Delta/Takip ID'sini okumak için.
DATA lv_maktx TYPE maktx.
DATA: gv_spras TYPE spras.

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

  gv_spras = sy-langu.




  SELECT tabname FROM zmatnr_tablolar                   "Aktif tabloları çekiyoruz.
                 INTO TABLE gt_active_tables
                 WHERE aktif = 'X'.

  IF gt_active_tables IS INITIAL.
    MESSAGE 'Aktif veri tablosu bulunamadı' TYPE 'I'.
    RETURN.
  ENDIF.

  LOOP AT gt_active_tables INTO gs_active_table.   "Aktif tabloları sırasıyla döngüye alıyoruz.

    CLEAR: lv_max_matnr,
           lv_last_processed_id,
           lv_max_id.

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

    SELECT MAX( zzid )                      "Döngüde okunan tablodan max zzid'yi çekiyoruz.
           FROM zpp_tbl_meo
           WHERE tanim = 'ZMMR_MEO_SQL'
           INTO @lv_last_processed_id.

    SELECT MAX( zzid )
           FROM (gs_active_table-tabname)  "Log tablosundan son zzid'yi çekiyoruz.
           INTO @lv_max_id.

    IF lv_max_id > lv_last_processed_id.         "Eğer raporun çalışmasından sonra kayıt varsa deltayı alıyoruz

      SELECT matnr, zzid FROM (gs_active_table-tabname)                 "
              INTO CORRESPONDING FIELDS OF TABLE @<gt_data>
              WHERE zzid > @lv_last_processed_id
              AND zzid <= @lv_max_id.


      DATA(lv_lines) = lines( <gt_data> ).                                 "Tablodaki kayıtları structure'a çekiyoruz
      WRITE: / 'İşlenecek kayıt sayısı' , lv_lines.
      LOOP AT <gt_data> INTO <gs_data>.

        CLEAR: lv_maktx.                                                    "Structure'dan gerekli alanları variable'lara çekiyoruz.
        ASSIGN COMPONENT 'MATNR' OF STRUCTURE <gs_data> TO <gv_matnr>.
        ASSIGN COMPONENT 'ZZID' OF STRUCTURE <gs_data> TO <gv_zzid>.

        IF <gv_matnr> IS ASSIGNED AND <gv_matnr> IS NOT INITIAL.
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

*&-----------------------------------------*---- SQL SÜRECİ  ------------------------------------------------------*

  DATA loop_final TYPE gs_final_package2.                       "SQL'den önce final paketinde gezip çalışıyor mu diye test et
  LOOP AT gt_final_package INTO loop_final.
    WRITE: / 'Materyal Adı' , loop_final-matnr.
    WRITE: / 'Materyal zzidsi' , loop_final-zzid.
    WRITE: / 'Materyal açıklaması' , loop_final-maktx.
  ENDLOOP.

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
