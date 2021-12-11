This the SQl code used in the project to pull data from the ELZ, its written in SQL

CREATE
    VIEW FTMPUB.TRANSACTION_RPT_V AS SELECT
        TXNS.ID AS OBJ_ID,
        TXNS.BATCH_ID,
        TXNS.TRANSMISSION_ID,
        TXNS.APP_ID,
        CAV.APP_NAME,
        CAV.APP_VERSION_ID,
        CAV.VERSION AS APP_VERSION,
        TXNS.type AS OBJ_TYPE,
        TXNS.STATUS,
        CLASS3.DESCRIPTION AS STATUS_DESC,
        TXNS.CREATED,
        TXNS.STATUS_DATE,
        (
            TXNS.STATUS_DATE - TXNS.CREATED
        ) AS STATUS_LATENCY,
        TXNS.MASTER_FLAG,
        TXNS.EVENT_ID,
        TXNS.OWNER_ID,
        TXNS.OBJ_CLASS,
        TXNS.SUBTYPE,
        CLASS2.DESCRIPTION AS SUBTYPE_DESC,
        TXNS.SENDER AS SENDER_ID,
        TXNS.RECEIVER AS RECEIVER_ID,
        TXNS.AUX_STATUS,
        PMTTXNS.PAYMENT_TYPE,
        PMTTXNS.PMC,
        PMTTXNS.BANK_CODE,
        BC.PNAME AS BANK_NAME,
        PMTTXNS.ACCOUNT AS ORIGINATOR_ACCOUNT,
        PMTTXNS.DEST_BANK_CODE,
        DBC.PNAME AS DEST_BANK_NAME,
        PMTTXNS.DEST_ACCOUNT AS RECIPIENT_ACCOUNT,
        (
            SELECT
                OV.SMALL_VALUE
            FROM
                OBJ_VALUE OV
            WHERE
                OV.OBJ_ID = TXNS.ID
                AND OV.CATEGORY = 'GPY_TXN_EXT'
                AND OV.KEY = 'RecipientName'
        ) AS RECIPIENT_NAME,
        (
            SELECT
                OV.SMALL_VALUE
            FROM
                OBJ_VALUE OV
            WHERE
                OV.OBJ_ID = TXNS.BATCH_ID
                AND OV.CATEGORY = 'GPY_BATCH_EXT'
                AND OV.KEY = 'CompanyName'
        ) AS ORIGINATOR_NAME,
        CASE
            WHEN TXNS.SUBTYPE IN(
                'IP_FROM_DBTR_RCL',
                'IP_FROM_CSM_RCL'
            ) THEN(
                SELECT
                    OV.SMALL_VALUE
                FROM
                    OBJ_VALUE OV
                WHERE
                    OV.OBJ_ID = TXNS.ID
                    AND OV.CATEGORY = 'StatusReason'
                    AND OV.KEY = 'ReasonCode'
            )
            ELSE(NULL)
        END AS REASON_CODE,
        PMTTXNS.AMOUNT,
        PMTTXNS.CURRENCY,
        PMTTXNS.FX_RATE,
        PMTTXNS.DEBIT_CREDIT_FLAG,
        PMTTXNS.TXN_TIMESTAMP,
        PMTTXNS.BOOK_DATE,
        PMTTXNS.VALUE_DATE,
        PMTTXNS.SETTLE_DATE,
        PMTTXNS.SETTLEMENT_SYSTEM,
        PMTTXNS.TXN_DATA1,
        PMTTXNS.TXN_DATA2,
        PMTTXNS.TXN_SEQUENCE,
        TRNSM.CHANNEL_ID,
        TRNSM.FILENAME,
        TRNSM.FILEADDRESS,
        CH.NAME AS CHANNEL_NAME,
        CH.DESCRIPTION AS CHANNEL_DESC,
        CH.INBOUND AS CHANNEL_DIR,
        CH.OPEN AS CHANNEL_OPEN,
        CH.Q_NAME AS CHANNEL_QNAME,
        CH.TIMEZONE AS CHANNEL_TIMEZONE,
        CH.LOCATION AS CHANNEL_LOCATION
    FROM
        (
            SELECT
                O.ID,
                T.BATCH_ID,
                T.TRANSMISSION_ID,
                O.APP_ID,
                O.type,
                O.STATUS,
                O.CREATED,
                O.STATUS_DATE,
                O.MASTER_FLAG,
                O.EVENT_ID,
                O.OWNER_ID,
                O.OBJ_CLASS,
                O.SUBTYPE,
                O.SENDER,
                O.RECEIVER,
                O.AUX_STATUS
            FROM
                TRANSACTION_BASE T
            JOIN OBJ_BASE O ON
                T.OBJ_ID = O.ID
        ) txns
    LEFT OUTER JOIN(
            SELECT
                T.OBJ_ID AS id,
                P.PAYMENT_TYPE,
                P.PMC,
                P.BANK_CODE,
                P.ACCOUNT,
                P.DEST_BANK_CODE,
                P.DEST_ACCOUNT,
                P.AMOUNT,
                P.CURRENCY,
                P.FX_RATE,
                P.DEBIT_CREDIT_FLAG,
                P.TXN_TIMESTAMP,
                P.BOOK_DATE,
                P.VALUE_DATE,
                P.SETTLE_DATE,
                P.SETTLEMENT_SYSTEM,
                T.TXN_DATA1,
                T.TXN_DATA2,
                T.TXN_SEQUENCE
            FROM
                TRANSACTION_BASE T
            JOIN OBJ_BASE O ON
                T.OBJ_ID = O.ID
            JOIN TXN_PAYMENT_BASE P ON
                T.OBJ_ID = P.OBJ_ID
        ) PMTTXNS ON
        TXNS.ID = PMTTXNS.ID
    LEFT OUTER JOIN(
            SELECT
                V.ID AS APP_VERSION_ID,
                A.NAME AS APP_NAME,
                A.ID AS APP_ID,
                V.VERSION
            FROM
                APP_VERSION V
            JOIN APPLICATION A ON
                V.APPLICATION_ID = A.ID
            WHERE
                V.EFFECTIVE_DATETIME =(
                    SELECT
                        MAX( EFFECTIVE_DATETIME )
                    FROM
                        APP_VERSION V2
                    WHERE
                        EFFECTIVE_DATETIME < FTM_TIMESTAMP()
                        AND A.ID = V2.APPLICATION_ID
                )
                AND DELETE_FLAG = 'N'
        ) CAV ON
        TXNS.APP_ID = CAV.APP_ID
    LEFT OUTER JOIN(
            SELECT
                o.id,
                o.app_id,
                pt.CHANNEL_ID,
                PT.FILENAME,
                PT.FILEADDRESS
            FROM
                TRANSMISSION_BASE PT
            JOIN OBJ_BASE O ON
                PT.OBJ_ID = O.ID
        ) trnsm ON
        TXNS.TRANSMISSION_ID = TRNSM.ID
    LEFT OUTER JOIN CHANNEL CH ON
        TRNSM.CHANNEL_ID = CH.ID
        AND CAV.APP_VERSION_ID = CH.APP_VERSION_ID
    LEFT OUTER JOIN PARTNER_GENERAL DBC ON
        PMTTXNS.DEST_BANK_CODE = DBC.PID
        AND DBC.VERSION = CURRENT_PARTNER_VERSION(DBC.PID)
    LEFT OUTER JOIN PARTNER_GENERAL BC ON
        PMTTXNS.BANK_CODE = BC.PID
        AND BC.VERSION = CURRENT_PARTNER_VERSION(BC.PID)
    LEFT OUTER JOIN CLASSIFICATION CLASS2 ON
        CLASS2.APP_VERSION_ID = CAV.APP_VERSION_ID
        AND CLASS2.CODE = TXNS.SUBTYPE
        AND CLASS2.SCHEME = 'OBJSUBTYPE_TXN'
    LEFT OUTER JOIN CLASSIFICATION CLASS3 ON
        CLASS3.APP_VERSION_ID = CAV.APP_VERSION_ID
        AND CLASS3.CODE = TXNS.STATUS
        AND CLASS3.SCHEME = 'OBJSTATUS_TXN'

