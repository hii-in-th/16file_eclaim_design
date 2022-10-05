# design

โครงสร้างชื่อไฟล์  
`{transectionId}_{filetype}`
- transectionId รหัสที่ระบบสร้างขึ้นมา
- filetype รูปแบบของไฟล์ 16 แฟ้ม INS, PAT, IPD, อื่นๆ  

example
```
sdfasd54sd654825_INS.txt
sdfasd54sd654825_ADP.txt
sdfasd54sd654825_CHA.txt
asasd54sd654826_INS.txt
asfasd54sd654826_ADP.txt
asfasd54sd654826_CHA.txt
```

```mermaid
journey
    title ลำดับการประมวลผล
    section ส่งข้อมูล eClaim
      รับข้อมูล แล้ว ทำการเข้าคิวรอประมวลผล: 3: ความง่าย
      ประมวลผลแยกเป็น 16 แฟ้ม:7: ความง่าย
      ส่งข้อมูลเข้าสู่ระบบ eClaim (Python): 2: ความง่าย
      บันทึกและรายงานผลการส่งออก: 5: ความง่าย
```
## รายละเอียดการประมวลผล  
```mermaid
stateDiagram
    direction LR
    [*] --> api : Restful json
    api --> Queue : store
    state Queue {
      direction LR
      RabbitMQ
      PostgreSQL
    }

    Queue --> HomewardProcessFiles
    state HomewardProcessFiles {
      direction LR
      ETL
      สร้างไฟล์ตามรูปแบบ
      transectionId_filetype
    }

    state send_eClaim {
        direction LR
        isShareProcess
        CallbackReport
    }

    HomewardProcessFiles --> send_eClaim
    
```
# ER

```mermaid
erDiagram
    homeward-eClaim ||--|{ common_wellnet_eclaim : "is"
    state ||--|| homeward-eClaim : "have"
    common_wellnet_eclaim {
        string transaction_id PK "ใช้ link กับข้อมูล"
        string hospital_id
        string person_id
        string ref_id
        string call_back_type "ตอนนี้ใช้เป็น homeward"
        string call_back_url "https://xxx.hii.in.th/yyy/zzz"
        json data
    }
    state {
        string transaction_id FK "ใช้ link กับข้อมูล"
        string status
        string message
        string errorMessage
        string taskProcess "แต่ละข้อความคั่นด้วย ,"
    }


```
# Flowchart

## api
```mermaid
flowchart LR
    
    subgraph Queue
        pgsql 
        RabbitMQ
    end

    subgraph api
        direction TB
        checkReq["เช็ค Require พื้นฐาน"]
        setStateQ["กำหนดสถานะ queuing"]
        genTran["สร้าง transactionId"]
        storeDB["บันทึกข้อมูลลง pgsql"]
        storeQ["บันทึกข้อมูลลง RabbitMQ"]

        checkReq --> setStateQ
        setStateQ --> genTran

        genTran --> storeDB
        storeDB --> storeQ
    end

     *input --> api
     api --->|"บันทึก"| Queue

```

# ETL Flow

```mermaid
flowchart LR
    subgraph Queue
        direction TB
        pgsql
        RabbitMQ
    end
    subgraph ETL
        input
        file1f[1.INS filter]
        file2f[2.PAT filter]
        fileNf[N.XXX filter]
        
        create1f["สร้าง 1.INS"]
        create2f["สร้าง 2.PAT"]
        createNf["สร้าง N.XXX"]


        input --> file1f
        input --> file2f
        input --> fileNf

        file1f --> create1f
        file2f --> create2f
        fileNf --> createNf
    end

    Queue -->|Streaming| input
```

# Main function send_eClaim Flow
```mermaid
    flowchart TB
    send_eClaim("ส่งเคลม(transactionId,username,password)")
    send_eClaim-->getFile[["ดึงรายการไฟล์ของ (transectionId)"]]
    getFile-->createBody[["สร้าง Body สำหรับส่งเคลม 16 แฟ้ม(List files)"]]
    createBody-->sendToNHSO[["ส่งข้อมูลเคลมไปยัง สปสช. (json body,username,password)"]]
    sendToNHSO-->updateStats[["รายงานบันทึกผลเคลม (transectionId, status)"]]
    updateStats-->moveArchive[["ย้ายไฟล์ไปยัง archive (transectionId)"]]
    moveArchive-->END(END)
```
## Sub function สร้าง Body สำหรับส่งเคลม 16 แฟ้ม  

```mermaid
    flowchart TB
    createBody("สร้าง Body สำหรับส่งเคลม 16 แฟ้ม (List files)")
    createBody-->fileSize[["ดูขนาดไฟล์ (List files)"]]
    fileSize-->base64[["ทำข้อมูลไฟล์ให้เป็น base64 (List files)"]]
    base64-->createJson[["สร้าง body json สำหรับส่งเคลม (List filesize,List base64)"]]
    createJson-->Return("Return (body json)")
```
## Sub function ส่งข้อมูลเคลมไปยัง สปสช.
```mermaid
    flowchart TB
    sendToNHSO("ส่งข้อมูลเคลมไปยัง สปสช. (json body,username,password)")
    sendToNHSO-->getToken[["authen เพื่อขอ token สำหรับส่งข้อมูล (username,password)"]]
    getToken-->send[["ส่งข้อมูลไปยัง eClaim สปสช.(json body,token)"]]
    send-->checkError{"ServerError หรือไม่"}
    checkError-->|"มี ให้ลอง 3 ครั้ง\nถ้าครบ ให้ throw LimitServerRetry"| send
    checkError-->|"ไม่มี Error"| Return("Return (ผลการส่ง)")
```
## Sub function รายงานบันทึกผลเคลม
```mermaid
    flowchart TB
    updateStats("รายงานบันทึกผลเคลม (transectionId, status)")
    updateStats-->|"อัพเดดข้อมูล\nstatus,message"| database[(Database\npgsql)]
    updateStats-->mainSystem[["รายงานผล\nไปยังระบบหลัก"]]
    database-->Return
    mainSystem-->Return
```
