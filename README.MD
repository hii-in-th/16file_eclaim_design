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

    Queue --> Process16files
    state Process16files {
      direction LR
      ETL
      สร้างไฟล์ตามรูปแบบ
      transectionId_filetype
    }
    Process16files --> send_eClaim
    send_eClaim --> report
    
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

## ETL Flow

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