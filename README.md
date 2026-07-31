# BDC-PROGRAM
Creating BDC Prog for creating material 

report ZGMK_BDC
       no standard page heading line-size 255.

*include bdcrecx1_s. (-)------------------------------------------------------> need to copy bdc declaration and performs in end of the include
DATA:   BDCDATA LIKE BDCDATA    OCCURS 0 WITH HEADER LINE. "+----------------->copy from include
data: it_raw type TRUXS_T_TEXT_DATA. "(+) declar it_raw it's mandatary filed in convert fm.
data: it_bms type table of bdcmsgcoll,  " (+) add this msg internal table drclaration
      wa_bms type bdcmsgcoll.  " (+) add this msg internal table drclaration
data : lv_msg type string.  " (+) add this lv_msg to store the return msg

*parameters: dataset(132) lower case.
PARAMETERS : p_file type localfile. " (+)-------------------------------------> remove old parameter / change old parameter data type

*data: begin of record, (-)--------------------------------------------------------------> this one structre with filling filed we need to convert this into one internal table
data: begin of record occurs 0, "(+)-----------------------------------------------------> Adding occurs 0 to convert into into table
* data element: MBRSH
        MBRSH_001(001),
* data element: MTART
        MTART_002(004),
* data element: XFELD
        KZSEL_01_003(001),
* data element: MAKTX
        MAKTX_004(040),
* data element: MEINS
        MEINS_005(003),
* data element: MATKL
        MATKL_006(009),
      end of record.
**(+)----------------------------declar types to receive the error msg.
      TYPES:BEGIN OF ty_error,
        line type i,
        msg type string,
        END OF ty_error.

        data: it_error type table of ty_error.
**(+)----------------------------declar types to receive the error msg.

** (+) add f4 help and selecting file for p_file and uploading file.
      at SELECTION-SCREEN ON VALUE-REQUEST FOR p_file.

        CALL FUNCTION 'F4_FILENAME'
         EXPORTING
           PROGRAM_NAME        = SYST-CPROG
*           DYNPRO_NUMBER       = SYST-DYNNR
*           FIELD_NAME          = ' '
         IMPORTING
           FILE_NAME           = p_file
                  .
start-of-selection.

CALL FUNCTION 'TEXT_CONVERT_XLS_TO_SAP'
  EXPORTING
*   I_FIELD_SEPERATOR          =
   I_LINE_HEADER              = 'X'
    i_tab_raw_data             = it_raw
   I_FILENAME                 =  p_file
*   I_STEP                     = 1
*   I_FILENAME_LONG            =
  tables
    i_tab_converted_data       = record[]
 EXCEPTIONS
   CONVERSION_FAILED          = 1
   OTHERS                     = 2
          .
** (+) add f4 help and selecting file for p_file and uploading file.


*perform open_dataset using dataset. (-)--------------------------------------- no need this just comment this lines
*perform open_group. (-)------------------------------------------------------- no need this just comment this lines

*do. (-)--------------------------------------- comment do loop and replace with loop at to move wa_record data into wa_bdcdata
loop at record. "(+)----------------------------add loop at record for data moving

*read dataset dataset into record. (-)--------------------------------------- no need this just comment this lines
if sy-subrc <> 0. exit. endif.
data(lv_index) = sy-tabix.  " (+) add this for check which line get error

perform bdc_dynpro      using 'SAPLMGMM' '0060'.
perform bdc_field       using 'BDC_CURSOR'
                              'RMMG1-MTART'.
perform bdc_field       using 'BDC_OKCODE'
                              'ENTR'.
perform bdc_field       using 'RMMG1-MBRSH'
                              record-MBRSH_001.
perform bdc_field       using 'RMMG1-MTART'
                              record-MTART_002.
perform bdc_dynpro      using 'SAPLMGMM' '0070'.
perform bdc_field       using 'BDC_CURSOR'
                              'MSICHTAUSW-DYTXT(01)'.
perform bdc_field       using 'BDC_OKCODE'
                              '=ENTR'.
perform bdc_field       using 'MSICHTAUSW-KZSEL(01)'
                              record-KZSEL_01_003.
perform bdc_dynpro      using 'SAPLMGMM' '4004'.
perform bdc_field       using 'BDC_OKCODE'
                              '=BU'.
perform bdc_field       using 'MAKT-MAKTX'
                              record-MAKTX_004.
perform bdc_field       using 'BDC_CURSOR'
                              'MARA-MATKL'.
perform bdc_field       using 'MARA-MEINS'
                              record-MEINS_005.
perform bdc_field       using 'MARA-MATKL'
                              record-MATKL_006.
*perform bdc_transaction using 'MM01'. (-)--------------------------------------- no need this just comment this lines
call TRANSACTION 'MM01' USING BDCDATA Mode 'N' MESSAGES INTO it_bms.  "(+)--------------------------------------- For calling we need to using this synatx.


**(+)--------------------------------------------> all msg are store it_bms we need error msg so loop error msg and convert into 1 full msg.
loop at it_bms into wa_bms WHERE MSGTYP = 'E' .
CALL FUNCTION 'FORMAT_MESSAGE'
 EXPORTING
*   ID              = SY-MSGID
*   LANG            = '-D'
   NO              = wa_bms-msgnr
   V1              = wa_bms-MSGV1
   V2              = wa_bms-MSGV2
   V3              = wa_bms-MSGV3
   V4              = wa_bms-MSGV4
 IMPORTING
   MSG             = lv_msg
 EXCEPTIONS
   NOT_FOUND       = 1
   OTHERS          = 2
          .
IF sy-subrc = 0.

append VALUE  ty_error( msg = lv_msg line = lv_index ) to it_error.

ENDIF.

endloop.

refresh: bdcdata, it_bms. "(+) clear the wa for next loop
clear wa_bms.
***------------------------------------------------------------------------------------- convert msg and clear bdcdata and it_bms.
*enddo. (-)--------------------------------------- replace enddo with endloop
ENDLOOP. "(+)--------------------------------------- add end loop.


**(+)-------------------------------------------------loop the error msg for display
LOOP AT it_error ASSIGNING FIELD-SYMBOL(<wa_error>).

  write:/ <wa_error>-line, <wa_error>-msg.

ENDLOOP.
**(+)-------------------------------------------------loop the error msg for display

*perform close_group. (-)------------------------------------------------------- no need this just comment this lines
*perform close_dataset using dataset. (-)--------------------------------------- no need this just comment this lines

*----------------------------------------------------------------------*
*      copy form bdc include                                         *
*----------------------------------------------------------------------*
FORM BDC_DYNPRO USING PROGRAM DYNPRO.
  CLEAR BDCDATA.
  BDCDATA-PROGRAM  = PROGRAM.
  BDCDATA-DYNPRO   = DYNPRO.
  BDCDATA-DYNBEGIN = 'X'.
  APPEND BDCDATA.
ENDFORM.

*----------------------------------------------------------------------*
*        copy form bdc include
*----------------------------------------------------------------------*
FORM BDC_FIELD USING FNAM FVAL.
*  IF FVAL <> NODATA.        (-)--------------------------------------- no need this just comment this lines
    CLEAR BDCDATA.
    BDCDATA-FNAM = FNAM.
    BDCDATA-FVAL = FVAL.
    APPEND BDCDATA.
*  ENDIF.                   (-)--------------------------------------- no need this just comment this lines
ENDFORM.
