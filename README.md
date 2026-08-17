*&---------------------------------------------------------------------*
*& Include          ZPPR_MEO_STOK_TAKIP_FUNC
*&---------------------------------------------------------------------*
*&---------------------------------------------------------------------*
*& Form get_data
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
form get_data .

  if r_kydt is not initial.

    select a~mblnr ,
           a~mjahr ,
           a~zeile ,
           a~matnr ,
           a~menge ,
           a~meins ,
           a~bwart ,
           a~cpudt ,
           a~budat ,
           b~mtart ,
           a~lifnr ,
           a~werks ,
           a~ebeln ,
           a~ebelp ,
           a~charg ,
           a~shkzg ,
           a~sjahr ,
           a~smbln ,
           a~smblp ,
           a~xblnr ,
           c~name1
      from matdoc as a
      inner join mara as b
      on b~matnr = a~matnr
      inner join lfa1 as c
      on c~lifnr = a~lifnr
      where a~matnr in @s_matnr
      and   a~budat in @s_bdt
      and   a~cpudt in @s_cpt
      and   b~mtart in @s_mtrt
      and   a~werks eq @p_wrk
      and   a~bwart in @s_bwrt
      into table @gt_mtdoc.

  endif.

endform.
*&---------------------------------------------------------------------*
*& Form disable_selection
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
form disable_selection .

  loop at screen.

    if r_kydt = 'X'.

      if screen-name cs 'P_DATE'
      or screen-name cs 'S_ID'.

        screen-input     = 0.
        screen-invisible = 1.
        screen-active    = 0.
      endif.

    elseif r_sil = 'X'.

      if screen-name cs 'S_MATNR'
      or screen-name cs 'S_BDT'
      or screen-name cs 'S_CPT'
      or screen-name cs 'S_MTRT'
      or screen-name cs 'P_WRK'
      or screen-name cs 'S_BWRT'
      or screen-name cs 'CB_AKTR'
      or screen-name cs 'CB_LOG'.

        screen-input     = 0.
        screen-invisible = 1.
        screen-active    = 0.
      endif.

    elseif r_srg = 'X'.

      if screen-name cs 'CB_LOG'
      or screen-name cs 'S_MTRT'
      or screen-name cs 'CB_AKTR'
      or screen-name cs 'P_DATE'.

        screen-input     = 0.
        screen-invisible = 1.
        screen-active    = 0.
      endif.

    endif.

    modify screen.

  endloop.

endform.
*&---------------------------------------------------------------------*
*& Form save_data
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
form save_data .

  clear: gt_ozet[], gt_detay[], gv_log_id.

  if cb_log is not initial and gt_mtdoc is not initial..

    perform save_log.

    loop at gt_mtdoc into gs_mtdoc.

*-- Özet tablo için Z'li tabloya kayıt atılır.
      clear gs_ozet.
      gs_ozet-zzid  = gv_log_id..
      gs_ozet-matnr = gs_mtdoc-matnr.
      gs_ozet-bwart = gs_mtdoc-bwart.
      gs_ozet-meins = gs_mtdoc-meins.
      gs_ozet-menge = gs_mtdoc-menge.
      gs_ozet-budat = gs_mtdoc-budat.
      gs_ozet-lifnr = gs_mtdoc-lifnr.

      gs_ozet-kayit_yapan  =  sy-uname.
      gs_ozet-log_trh      =  sy-datum.
      gs_ozet-log_saat     =  sy-uzeit.

*-- İade / çıkış hareketlerinde miktarı ters çevirmek istenirse;
      if gs_mtdoc-shkzg = 'H'.
        gs_ozet-menge = gs_ozet-menge  * -1.
      endif.

      collect gs_ozet into gt_ozet.

*-- Detay tablo için Z'li tabloya kayıt atılır.
      clear gs_detay.
      gs_detay-zzid     =  gv_log_id." ilgili kayıtlara id verilir.
      gs_detay-mblnr    =  gs_mtdoc-mblnr.
      gs_detay-mjahr    =  gs_mtdoc-mjahr.
      gs_detay-zeile    =  gs_mtdoc-zeile.
      gs_detay-matnr    =  gs_mtdoc-matnr.
      gs_detay-menge    =  gs_mtdoc-menge.
      gs_detay-meins    =  gs_mtdoc-meins.
      gs_detay-bwart    =  gs_mtdoc-bwart.
      gs_detay-cpudt    =  gs_mtdoc-cpudt.
      gs_detay-budat    =  gs_mtdoc-budat.
      gs_detay-lifnr    =  gs_mtdoc-lifnr.
      gs_detay-werks    =  gs_mtdoc-werks.
      gs_detay-ebeln    =  gs_mtdoc-ebeln.
      gs_detay-ebelp    =  gs_mtdoc-ebelp.
      gs_detay-charg    =  gs_mtdoc-charg.
      gs_detay-shkzg    =  gs_mtdoc-shkzg.
      gs_detay-sjahr    =  gs_mtdoc-sjahr.
      gs_detay-smbln    =  gs_mtdoc-smbln.
      gs_detay-smblp    =  gs_mtdoc-smblp.
      gs_detay-xblnr    =  gs_mtdoc-xblnr.
      gs_detay-kayit_yapan  =  sy-uname.
      gs_detay-kayit_trh    =  sy-datum.
      gs_detay-kayit_saat   =  sy-uzeit.

      append gs_detay to gt_detay.

    endloop.

    modify zppt_meo_stk_ozt from table gt_ozet.

    if sy-subrc <> 0.
      message 'Log özet kaydı oluşturulamadı.' type 'E'.
    endif.

    modify zppt_meo_stok from table gt_detay.

    if sy-subrc <> 0.
      message 'Log detay kaydı oluşturulamadı.' type 'E'.
    endif.

  endif.

endform.
*&---------------------------------------------------------------------*
*& Form save_log
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
form save_log .

  clear gv_log_id.

  perform get_next_log_id changing gv_log_id.

  get time.
  gv_bit_tr   = sy-datum.
  gv_bit_saat = sy-uzeit.

  "------------------------------------------------------------
  " Başlık kaydı
  "------------------------------------------------------------
  clear gs_log_h.

  gs_log_h-zzid     = gv_log_id.
  gs_log_h-tanim    = 'GR_M_MB51'.
  gs_log_h-bas_tr   = gv_bas_tr.
  gs_log_h-bas_saat = gv_bas_saat.
  gs_log_h-bit_tr   = gv_bit_tr.
  gs_log_h-bit_saat = gv_bit_saat.
  gs_log_h-ernam    = sy-uname.
  gs_log_h-loekz    = space.

  insert zpp_tbl_meo from gs_log_h.

  if sy-subrc <> 0.
    message 'Log başlık kaydı oluşturulamadı.' type 'E'.
  endif.

  commit work and wait.

  message | Log kaydı oluşturuldu. ID: { gv_log_id }| type 'S'.

endform.
*&---------------------------------------------------------------------*
*& Form set_log_time
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
form set_log_time .

  get time.
  gv_bas_tr    = sy-datum.
  gv_bas_saat  = sy-uzeit.

endform.
*&---------------------------------------------------------------------*
*& Form get_next_log_id
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*&      <-- GV_LOG_ID
*&---------------------------------------------------------------------*
form get_next_log_id  changing cv_zzid type zpp_tbl_meo-zzid.

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
    message 'Log ID için numara aralığından değer alınamadı.' type 'E'.
  endif.

  call function 'CONVERSION_EXIT_ALPHA_INPUT'
    exporting
      input  = lv_number
    importing
      output = cv_zzid.

endform.
*&---------------------------------------------------------------------*
*& Form data_log
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
form data_log .

  clear: gt_log[].
  if r_srg is not initial.

    select *
      from zppt_meo_stok
      into table @gt_log
      where zzid in @s_id
      and   matnr in @s_matnr
      and   budat in @s_bdt
      and   cpudt in @s_cpt
      and   werks eq @p_wrk
      and   bwart in @s_bwrt.

  endif.
endform.
*&---------------------------------------------------------------------*
*& Form delete_data
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
form delete_data .

  if r_sil is not initial.

    types: begin of ty_zzid,
             zzid type zpp_tbl_meo-zzid,
           end of ty_zzid.

    data: lt_zzid       type standard table of ty_zzid,
          ls_zzid       type ty_zzid,
          lv_head_count type c,
          lv_upd_count  type c,
          lv_det_count  type char4,
          lv_answer     type c.

    clear: lt_zzid,
           lv_head_count,
           lv_upd_count,
           lv_det_count,
           lv_answer.

    "------------------------------------------------------------
    " En az bir kriter zorunlu:
    " S_ID veya P_DATE
    "------------------------------------------------------------
    if s_id[] is initial and p_date is initial.

      message
      'Log ID veya silme tarihi alanlarından en az biri dolu olmalıdır.'
                                              type 'S' display like 'E'.
      return.

    endif.

    "------------------------------------------------------------
    " Sadece bu rapora ait loglar silinmeli
    " TANIM = GR_M_MB51
    " 1) Sadece S_ID doluysa: o ID'ler
    " 2) Sadece P_DATE doluysa: BAS_TR =< P_DATE
    " 3) İkisi de doluysa: ID + tarih kesişimi
    "------------------------------------------------------------
    if s_id[] is not initial and p_date is initial.

      select zzid
        from zpp_tbl_meo
        into table lt_zzid
        where zzid  in s_id
        and tanim eq 'GR_M_MB51'
        and loekz eq space.

    elseif s_id[] is initial and p_date is not initial.

      select zzid
        from zpp_tbl_meo
        into table lt_zzid
        where bas_tr le p_date
        and tanim  eq 'GR_M_MB51'
        and loekz  eq space.

    else.

      select zzid
        from zpp_tbl_meo
        into table lt_zzid
        where zzid   in s_id
        and bas_tr le p_date
        and tanim  eq 'GR_M_MB51'
        and loekz  eq space.

    endif.

    if lt_zzid[] is initial.

      message
      'Silme kriterlerine uygun GR_M_MB51 Log kaydı bulunamadı.'
                                       type 'S' display like 'E'.
      return.

    endif.

    sort lt_zzid by zzid.
    delete adjacent duplicates from lt_zzid comparing zzid.

    describe table lt_zzid lines lv_head_count.

    "------------------------------------------------------------
    " Dialog çalışıyorsa onay al
    "------------------------------------------------------------
    if sy-batch is initial.

      data : txt type char250.

      concatenate lv_head_count ' adet GR_M_MB51 '
      ' Log başlığı silindi olarak işaretlenecek'
      ' ve detaylar bilgileri silinecek, devam edilsin mi? ' into txt.


      call function 'POPUP_TO_CONFIRM'
        exporting
          titlebar              = 'LOG Silme Onayı'
          text_question         = txt
          text_button_1         = 'Evet'
          text_button_2         = 'Hayır'
          default_button        = '2'
          display_cancel_button = 'X'
        importing
          answer                = lv_answer
        exceptions
          others                = 1.

      if lv_answer ne '1'.
        message 'Silme işlemi iptal edildi.' type 'S'.
        return.
      endif.

    endif.

    "------------------------------------------------------------
    " Detay Tablosu fiziksel silinir.
    " Başlık Tablosuna silme işareti atılır.
    " Ek güvenlik: UPDATE sırasında da TANIM kontrol edilir.
    "------------------------------------------------------------
    loop at lt_zzid into ls_zzid.

      delete from zppt_meo_stk_ozt where zzid = ls_zzid-zzid.

      delete from zppt_meo_stok where zzid = ls_zzid-zzid.

      lv_det_count = lv_det_count + sy-dbcnt.

      update zpp_tbl_meo
      set loekz   = 'X'
      ernam_del   = sy-uname
      datum_del   = sy-datum
      where zzid  = ls_zzid-zzid
      and tanim   = 'GR_M_MB51'
      and loekz   = space.

      lv_upd_count = lv_upd_count + sy-dbcnt.

    endloop.

    commit work and wait.

    clear txt.

    concatenate lv_upd_count ' adet GR_M_MB51 '
    ' Log başlığı silindi olarak işaretlendi, '
    lv_det_count ' detay kaydı silindi.' into txt.

    message txt type 'S'.

  endif.

endform.
*&---------------------------------------------------------------------*
*& Form data_aktar
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
form data_aktar .

  if cb_aktr is not initial.

    perform start_of_conn.
    check lv_stop is initial .
    perform insert_values_to_ex_sql.
    perform end_of_conn.

  endif.

endform.
*&---------------------------------------------------------------------*
*& Form start_of_conn
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
form start_of_conn .

  clear: dbs, lv_stop.

  case sy-sysid.
    when 'O4P'.
      dbs =  dbs_prod .
    when 'O4D' or 'O4Q'.
      dbs = dbs_test.
    when others.
      lv_stop = 'X' .
      message |{ sy-sysid } sistemi için DBCON bağlantısı tanımlanmamış|
      type 'I'.
      return.
  endcase.

  try.
      exec sql.

        CONNECT TO :dbs AS 'c1'

      endexec.
                                                        "#EC CI_EXECSQL
                                                        "#EC CI_EXECSQL
      exec sql.

        SET CONNECTION 'c1'

      endexec.
                                                        "#EC CI_EXECSQL

    catch cx_sy_native_sql_error into exc_ref.
      error_text = exc_ref->get_text( ).
      lv_stop = 'X' .

      message error_text type 'I' .
  endtry .
                                                        "#EC CI_EXECSQL
endform.
*&---------------------------------------------------------------------*
*& Form insert_values_to_ex_sql
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
form insert_values_to_ex_sql .

  data: exc_ref    type ref to cx_sy_native_sql_error,
        error_text type string,
        ls_sas_flg type xfeld.


  select zzid,
         matnr,
         bwart,
         meins,
         budat,
         lifnr,
         menge
    from zppt_meo_stk_ozt
    where matnr in @s_matnr
    and budat in @s_bdt
    and bwart in @s_bwrt
    into corresponding fields of table @gt_aktar.

  if gt_aktar[] is initial.
    message 'Aktarılacak veri bulunamadı' type 'S' display like 'W'.
    return.
  endif.

  " ---- SQL tarafına aktarım ----
  try.

      loop at gt_aktar into gs_aktar.

        " Aynı anahtara sahip kayıt varsa önce sil
        "(kayıt yoksa hiçbir şey silinmez)
                                                        "#EC CI_EXECSQL
        exec sql.
          DELETE FROM dbo.t_mb51
          WHERE zzid  = :gs_aktar-zzid
          AND   matnr = :gs_aktar-matnr
          AND   bwart = :gs_aktar-bwart
          AND   meins = :gs_aktar-meins
          AND   budat = :gs_aktar-budat
          AND   lifnr = :gs_aktar-lifnr
        endexec.
                                                        "#EC CI_EXECSQL

        " Koşulsuz ekle
                                                        "#EC CI_EXECSQL
        exec sql.
          INSERT INTO dbo.t_mb51
          ( zzid, matnr, bwart, meins, budat, lifnr, menge )
          VALUES
          ( :gs_aktar-zzid,
            :gs_aktar-matnr,
            :gs_aktar-bwart,
            :gs_aktar-meins,
            :gs_aktar-budat,
            :gs_aktar-lifnr,
            :gs_aktar-menge )
        endexec.
                                                        "#EC CI_EXECSQL

        update zppt_meo_stk_ozt set aktarildi = 'X'
        where zzid  = gs_aktar-zzid
          and matnr = gs_aktar-matnr
          and bwart = gs_aktar-bwart
          and meins = gs_aktar-meins
          and budat = gs_aktar-budat
          and lifnr = gs_aktar-lifnr.

      endloop.

      " ---- Tüm loop hatasız bittiyse COMMIT ----
      " Harici SQL bağlantısı için commit
                                                        "#EC CI_EXECSQL
      exec sql.
        COMMIT
      endexec.
                                                        "#EC CI_EXECSQL

      commit work.

    catch cx_sy_native_sql_error into exc_ref.
                                                        "#EC CI_EXECSQL
      exec sql.
        ROLLBACK
      endexec.
                                                        "#EC CI_EXECSQL
      " SAP tarafı update'lerini geri al
      rollback work.

      error_text = exc_ref->get_text( ).
      message error_text type 'I'.
      ls_sas_flg = 'X'.

  endtry.

  " ---- 4) Sonuç mesajı ----
  if ls_sas_flg is initial.
    message 'Aktarım İşlemi Tamamlandı' type 'S' display like 'I'.
  endif.

endform.
*&---------------------------------------------------------------------*
*& Form end_of_conn
*&---------------------------------------------------------------------*
*& text
*&---------------------------------------------------------------------*
*& -->  p1        text
*& <--  p2        text
*&---------------------------------------------------------------------*
form end_of_conn .

                                                        "#EC CI_EXECSQL
  exec sql.

    COMMIT WORK AND WAIT.
    DISCONNECT 'c1'

  endexec.
                                                        "#EC CI_EXECSQL
endform.
