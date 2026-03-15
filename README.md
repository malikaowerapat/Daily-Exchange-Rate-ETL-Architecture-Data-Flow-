# 🏦 Daily Exchange Rate ETL Dataflow from API(Bank of Thailand) to Sharepoint

💰This project is a Python script designed to extract daily exchange rate data, unzip compressed files, read the CSV files inside, and transform data from multiple formats into a single standardized structure. It prioritizes data sources based on a defined hierarchy, then generates an Exchange Rate Master table with four columns: date, from_currency, to_currency, and rate. The final output is saved as a CSV file, and each execution is logged separately in a text file. Overall, this code functions as a small ETL system for daily exchange rates, with the goal of producing a clean, ready-to-use output file that can be integrated into other systems such as data imports, ERP, BI, automation, or downstream integrations.



โปรเจกต์นี้เป็นสคริปต์ Python สำหรับ ดึงข้อมูลอัตราแลกเปลี่ยนรายวัน, แตกไฟล์ ZIP, อ่านไฟล์ CSV ภายใน, แปลงข้อมูลจากหลายรูปแบบให้เป็นมาตรฐานเดียวกัน, เลือกแหล่งข้อมูลตามลำดับความสำคัญ, จากนั้นสร้าง ตาราง Exchange Rate Master แบบ 4 คอลัมน์ (date, from_currency, to_currency, rate) และบันทึกผลลัพธ์ลง CSV พร้อมเขียน log การรันทุกครั้งไว้ในไฟล์ข้อความแยกต่างหาก โค้ดนี้ทำระบบ ETL ขนาดเล็กสำหรับ Exchange Rate รายวัน โดยเน้นให้ได้ไฟล์ผลลัพธ์ที่พร้อมใช้งานต่อในระบบอื่น เช่น Data import, ERP, BI, automation หรือ integration ต่อไป
## 1) ดาวน์โหลดไฟล์ ZIP ของอัตราแลกเปลี่ยน

สคริปต์จะพยายามดาวน์โหลดไฟล์ `ER_CSV_EN.zip` จากเว็บไซต์ต้นทางผ่านทั้ง `https` และ `http` โดยมี retry mechanism และ fallback กรณี SSL verify มีปัญหา เพื่อให้การดาวน์โหลดมีความเสถียรมากขึ้น

## **2) แตก ZIP และค้นหาไฟล์ CSV ภายใน**

เมื่อดาวน์โหลดเสร็จ สคริปต์จะ extract ไฟล์ ZIP ไปยังโฟลเดอร์ชั่วคราว และ list ไฟล์ `.csv` ทั้งหมดที่อยู่ภายใน เพื่อเตรียมเลือกไฟล์ตามวันที่ที่ต้องการใช้งาน

## 3) เลือก “วันที่เป้าหมาย” ของข้อมูล

ระบบจะพยายามเลือกไฟล์ของ **วันปัจจุบัน (Asia/Bangkok)** ก่อน ถ้าไม่มีไฟล์ของวันนี้ จะ fallback ไปใช้ **วันที่ล่าสุดที่มีอยู่ใน ZIP** แทน เพื่อไม่ให้กระบวนการล้มเหลวเพราะข้อมูลยังไม่ออกในวันนั้น

## 4) อ่าน CSV แบบยืดหยุ่น

แต่ละไฟล์ CSV ถูกอ่านด้วย logic ที่ค่อนข้างทนทาน เช่น:

แต่ละไฟล์ CSV ถูกอ่านด้วย logic ที่ค่อนข้างทนทาน เช่น:

รองรับหลาย encoding (`utf-8-sig`, `utf-8`, `cp1252`, `latin1`)

ค้นหา header row จากข้อความที่มีคำว่า `country` และ `currency` แทนการสมมุติว่าหัวตารางต้องอยู่บรรทัดแรกเสมอ

รองรับทั้ง delimiter แบบ comma และรูปแบบพิเศษตามที่พบในไฟล์ต้นทาง

## 4)  Normalize ข้อมูลจาก 2 รูปแบบตาราง

สคริปต์รองรับการ parse ตารางแลกเปลี่ยนอย่างน้อย 2 รูปแบบหลัก:

csv1_avg_counter_rates

csv2_lseg_bkk_crossing

ทั้งสองแบบจะถูกแปลงให้กลายเป็น schema เดียวกัน เช่น:

`datecountrycurrencyunitbuy_rawsell_rawthb_per_1_buythb_per_1_sellsource_tablesource_file`

- `date`
- `country`
- `currency`
- `unit`
- `buy_raw`
- `sell_raw`
- `thb_per_1_buy`
- `thb_per_1_sell`
- `source_table`
- `source_file`

## 6)   คำนวณอัตรา THB ต่อ 1 หน่วยสกุลเงิน

โค้ดจะคำนวณ `thb_per_1_buy` และ `thb_per_1_sell` โดยเอา `buy_raw` และ `sell_raw` หารด้วย `unit` ที่ parse มาจากชื่อประเทศ เช่นกรณีมีข้อความประเภท `(100)` หรือ `(1,000)` ใน field ประเทศ เพื่อ normalize ให้เหลือค่า “ต่อ 1 หน่วยสกุลเงินจริง

## 7)   รวมข้อมูลจากหลายแหล่งแล้วเลือกแหล่งที่สำคัญกว่า

ถ้าวันเดียวกัน / currency เดียวกัน ถูกพบจากหลาย source โค้ดจะเลือกข้อมูลจาก source ที่มี priority สูงกว่า โดยกำหนดให้:

- `csv1_avg_counter_rates` มี priority สูงกว่า
- `csv2_lseg_bkk_crossing` เป็น fallback รองลงมา

## 8) สร้าง Exchange Rate Master แบบ 4 คอลัมน์

จากฐานข้อมูล normalized แล้ว สคริปต์จะสร้าง output สุดท้ายในรูป:

- `date`
- `from_currency`
- `to_currency`
- `rate`

## 9) บันทึกผลลัพธ์เป็น 2 ไฟล์

ผลลัพธ์สุดท้ายถูกบันทึกออกมาเป็นเพียง 2 ไฟล์:

- `DailyExchangeMaster.csv` สำหรับข้อมูล exchange rate หลัก
- `log_daily_exchange.txt` สำหรับบันทึกผลการรัน, status, duration, จำนวน currency, USD->THB และประวัติการรันย้อนหลัง
