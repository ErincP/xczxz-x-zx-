*&---------------------------------------------------------------------*
*& Report ZMMR_MEO_SQL_TRANSFER
*&---------------------------------------------------------------------*
*&Yazan : Erinç Pırıldar
*& Tarih : 12.08.2026
*& CNo   :
*& TCode :
*& Amaç  : MEO veri çekme programları envanterinden çekilen verileri SQL'e transfer etme.
*&---------------------------------------------------------------------*
report zmmr_meo_sql.

**********************************************************************
*---- VERİ TANIMLARI -----*

data: begin of gs_active_table,                      " Tablo listesinde dönecek structure
        tabname type zmatnr_tablolar-tabname,
      end of gs_active_table.

data: gt_active_tables like table of gs_active_table.   "Tablo listesini tutacak geçici tablo yapısı


data: dref_tab  type ref to data,
      dref_line type ref to data.

data: begin of gs_final_package,                "döngüde tablolardan alınan verilerin toplanacağı structure
        matnr type matnr,
        zzid  type zpp_meo_id,
        maktx type maktx,
      end of gs_final_package.

data: gt_final_package like table of gs_final_package.   " Final tablosu

data: gv_last_processed_id type zpp_tbl_meo-zzid,   " Delta takibini yapmaya yarayacak variable'lar
      gv_max_id            type zpp_tbl_meo-zzid.

types: begin of ty_makt_map,
         matnr type matnr,
         maktx type maktx,
       end of ty_makt_map.

data: gt_makt_map type hashed table of ty_makt_map with unique key matnr,   " Açıklamaları tutacak geçici tablo
      gs_makt_map type ty_makt_map.

types: begin of gs_final_package2,                "TEST AMAÇLI
         matnr type matnr,
         zzid  type zpp_meo_id,
         maktx type maktx,
       end of gs_final_package2.

field-symbols: <gt_data>  type standard table,        " Dinamik veri yapısı için alan sembolleri
               <gs_data>  type any,
               <gv_matnr> type any,
               <gv_zzid>  type any, "Delta/Takip ID'sini okumak için.
               <fs_pkg>   like line of gt_final_package.

data lv_maktx type maktx.         " Malzeme tanımı için

data: gv_spras type spras.        " Dil için.

data: gv_log_id           type zpp_tbl_meo-zzid,   " LOG süreci için global değişkenler
      gv_baslangic_saati  type uzeit,
      gv_bitis_saati      type uzeit,
      gv_baslangic_tarihi type datum,
      gv_bitis_tarihi     type datum,
      gv_tanim            type zpp_meo_tanim,
      meo_log             type zpp_tbl_meo.


********************************************************************** ANA YÜRÜTME BLOĞU

start-of-selection.

  perform set_log_time.          "Programı çalıştırmadan evvel başlangıç tarihini ve saatini al

  gv_spras = sy-langu.           " Dil.



  select max( zzid )                      "Log tablosundan, bu raporun son çalıştırıldığı ID'yi çekiyoruz.
           from zpp_tbl_meo
           where tanim = 'ZMMR_MEO_SQL'
           into @gv_last_processed_id.
    
  if gv_last_processed_id  is initial.
    gv_last_processed_id = 0.
  endif.

  select max( zzid )                      " Log tablosundan, kayıt yapan son ID'yi çekiyoruz.
    from zpp_tbl_meo                                       "(sistemin ulaştığı son nokta)
    into @gv_max_id.

  select tabname from zmatnr_tablolar                   "Aktif tabloları çekiyoruz.
                    into table gt_active_tables
                    where aktif = 'X'.

  if gt_active_tables is initial.
    message 'Aktif veri tablosu bulunamadı' type 'I'.
    return.
  endif.

  if gv_max_id > gv_last_processed_id.        " Bizden sonra yeni hareket var mı?

    loop at gt_active_tables into gs_active_table.            "Varsa aktif tabloları sırasıyla döngüye alıyoruz.

      write: / 'İşlenen Tablo: ', gs_active_table-tabname.


      try.
          create data dref_tab type table of (gs_active_table-tabname).   "Dinamik bellek alanlarını oluşturuyoruz.
          assign dref_tab->* to <gt_data>.

          create data dref_line type (gs_active_table-tabname).
          assign dref_line->* to <gs_data>.

        catch cx_sy_create_data_error.
          write: / 'HATA: Tablo yapısı dinamik olarak oluşturulamadı!'.
          continue.
      endtry.


      select matnr, zzid from (gs_active_table-tabname)           " Sıradaki tablodan delta aralığını çekiyoruz
              into corresponding fields of table @<gt_data>
              where zzid > @gv_last_processed_id
              and zzid <= @gv_max_id.


      data(lv_lines) = lines( <gt_data> ).
      write: / |-> Çekilen kayıt sayısı : { lv_lines }|.



      loop at <gt_data> into <gs_data>.       "Dinamik tablodaki kayıtlar içerisinde structure ile dönüyoruz.

        clear: lv_maktx.                                                    "Structure'dan gerekli alanları variable'lara çekiyoruz.
        assign component 'MATNR' of structure <gs_data> to <gv_matnr>.
        assign component 'ZZID' of structure <gs_data> to <gv_zzid>.




        if <gv_matnr> is assigned and <gv_matnr> is not initial and    " Eğer matnr ve zzid alanları boş değilse
           <gv_zzid> is assigned and <gv_zzid> is not initial.
          clear gs_final_package.
          gs_final_package-matnr = <gv_matnr>.                      " Bu satırı SQL'e gidecek olan tabloya atıyoruz.
          gs_final_package-zzid = <gv_zzid>.
          append gs_final_package to gt_final_package.
        endif.


      endloop.

      clear: dref_tab, dref_line.
      unassign: <gt_data>, <gs_data>, <gv_zzid>, <gv_matnr>.

    endloop.    "Tablo döngüsünün sonu






    if gt_final_package is not initial.    "Eğer final paketi bomboş değilse



      sort gt_final_package by matnr ascending    " Matnr'leri alt alta, zzid'lerine göre sırala
                               zzid descending.

      delete adjacent duplicates from gt_final_package.   " Eski zzid'ye sahip olanları sil




      select matnr, maktx                        " gt_final_package'da olan malzemeleri makt içerisinde bul ve bu malzemelerin adı ve tanımlarını al
        from makt
        for all entries in @gt_final_package
        where matnr = @gt_final_package-matnr
        and spras = @gv_spras
        into table @gt_makt_map.



      loop at gt_final_package assigning <fs_pkg>.    "gt_final_package içinde gez
        read table gt_makt_map into gs_makt_map      "Malzeme açklamaları tablosunda gt_final_package'daki malzeme adları ile arama yap
             with key matnr = <fs_pkg>-matnr.

        if sy-subrc = 0.                              " Eğer bir malzemeyi gt_makt_map tablosunda bulduysa
          <fs_pkg>-maktx = gs_makt_map-maktx.       " Onun açıklamasını alıp gt_final_package tablosuna ekle
        endif.
      endloop.
      unassign <fs_pkg>.



      perform save_log.   "İşlem sonunda log at


    else.
      write: / , / 'Tablolarda belirtilen delta aralığına uygun kayıt bulunamadı.'.
    endif.



  else.
    write: /, / 'Yeni delta kaydı bulunamadı. Sistem güncel.'.
  endif.


********************************************************************** ANA YÜRÜTME BLOĞU SONU



**********************************************************************
*SQL SÜRECİ


  data loop_final type gs_final_package2.                       "SQL'den önce final paketinde gezip çalışıyor mu diye test et
  loop at gt_final_package into loop_final.
    write: / 'Materyal Adı' , loop_final-matnr.
    write: / 'Materyal zzidsi' , loop_final-zzid.
    write: / 'Materyal açıklaması' , loop_final-maktx.
  endloop.


*& SQL SÜRECİ  ------------------------------------------------------


*&-----------------------------------------*---- LOG SÜRECİ İÇİN FORMLAR




form set_log_time.    "Log için başlangıç ve bitiş saati ve tarihi alma
  get time.
  gv_baslangic_tarihi = sy-datum.
  gv_baslangic_saati = sy-uzeit.
endform.


"Log için sıradaki ID'yi al
form get_next_log_id changing cv_zzid type zpp_tbl_meo-zzid.
  data: lv_number type char10.
  clear: cv_zzid, lv_number.
  call function 'NUMBER_GET_NEXT'
    exporting
      nr_range_nr             = '1'
      object                  = 'ZPP_MEO_BS'
    importing
      number                  = lv_number
    exceptions
      interval_not_found      = 1
      number_range_not_intern = 2
      object_not_found        = 3
      quantity_is_0           = 4
      quantity_is_not_1       = 5
      interval_overflow       = 6
      buffer_overflow         = 7
      others                  = 8.
  if sy-subrc <> 0.
    message 'Log ID için numara aralığından değer alınmadı.' type 'E'.
  endif.

  call function 'CONVERSION_EXIT_ALPHA_INPUT'
    exporting
      input  = lv_number
    importing
      output = cv_zzid.
endform.




form save_log.                              " Log alıyoruz.
  clear gv_log_id.

  perform get_next_log_id changing gv_log_id.

  get time.
  gv_bitis_tarihi = sy-datum.
  gv_bitis_saati = sy-uzeit.

  clear meo_log.

  meo_log-zzid = gv_log_id.
  meo_log-tanim = 'ZMMR_MEO_SQL'.
  meo_log-bas_tr = gv_baslangic_tarihi.
  meo_log-bas_saat = gv_baslangic_saati.
  meo_log-bit_tr = gv_bitis_tarihi.
  meo_log-bit_saat = gv_bitis_saati.
  meo_log-ernam = sy-uname.
  meo_log-loekz = space.

  insert zpp_tbl_meo from meo_log.

  if sy-subrc <> 0.
    message 'Log başlık kaydı oluşturulamadı' type 'E'.
  endif.

  commit work and wait.
  message |Log Kaydı Oluşturuldu. ID : { gv_log_id }| type 'S'.

endform.


*&-----------------------------------------*---- LOG SÜRECİ  ------------------------------------------------------*
