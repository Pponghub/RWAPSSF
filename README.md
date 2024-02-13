1.แก้ปัญหา front - running ด้วย commit - reveal
- เพิ่มการใส่ salt ลงในขั้นตอน input เพื่อที่จะ hash คำตอบของผู้เล่น ทำให้คนอื่นไม่รู้ว่าผู้เล่นเลือก choice อะไร
- เพิ่มการ revealPlayer โดยผู้เล่นทั้ง 2 คนต้องมากด reveal ก่อนเพื่อยืนยันการเลือก เกมจึงจะจบลงได้

2.แก้ปัญหาเงิน ETH ถอนไม่ได้ เมื่อลงเงินแล้วไม่มีผู้เล่นคนที่ 2 หรือผู้เล่นคนที่ 2 ไม่ยอมส่ง input
ผมสร้าง function withdraw เพื่อในกรณีที่เกินเวลาที่กำหนดไปแล้ว สามารถกดถอนเงินที่ลงไปคืนได้
- ถ้ามีผู้เล่นเพียงคนเดียว เมื่อเกิน 10 นาที จะกดปุ่ม withdraw เพื่อถอนเงินออกมาได้
- ถ้ามีผู้เล่น 2 คน แต่มีเพียงคนเดียวที่ใส่ input เข้าไป (commit) เมื่อเกิน 5 นาที  คนที่ใส่ input ไปแล้วจะสามารถกดปุ่ม withdraw เพื่อรับรางวัลทั้งหมดที่อยู่ในกองกลางได้
- ถ้ามีผู้เล่น 2 คน แต่มีเพียงคนเดียวที่ส่ง reveal เมื่อเกิน 5 นาที คนที่ส่ง reveal ไปแล้วจะสามารถกดปุ่ม withdraw เพื่อรับรางวัลทั้งหมดที่อยู่ในกองกลางได้

```
function withdraw(uint idx) public {
        address payable account = payable(player[idx].addr);
        if(numPlayer == 1 ){
            require((block.timestamp - player[idx].time) > 600 ,"not enough time 1");
            account.transfer(reward);
        }else if(numPlayer == 2 && numCommit==1){
            if(idx == 0){
                require((block.timestamp - commitTimeP0) > 300,"not enough time 2 ");
            }else{
                require((block.timestamp - commitTimeP1) > 300,"not enough time 3 ");
            }
            account.transfer(reward);
        }else if(numPlayer == 2 && numReveal==1){
            if(idx == 0){
                require((block.timestamp - revealTimeP0) > 300,"not enough time 4 ");
            }else{
                require((block.timestamp - revealTimeP1) > 300,"not enough time 5 ");
            }
            account.transfer(reward);
        }
    }
```

3.แก้ปัญหาแยกไม่ออกว่าเราเป็น idx เท่าไหร่
- เปลี่ยนไปใช้ address ในการระบุตัวตนเเทน ผู้เล่นไม่ต้องจำแล้วว่าเป็น idx เท่าไหร่

4.ทำให้เล่นได้หลายๆรอบโดยไม่ต้อง deploy ใหม่
- เพิ่ม function reset เพื่อทำให้เกมเริ่มใหม่ได้ทันทีโดยไม่ต้อง deploy โดยเมื่อจบขั้นตอนการจ่ายเงิน ETH ในกรณีใดก็ตาม (กด withdraw , ชนะ/แพ้/เสมอ) เกมจะใช้ฟังก์ชั่น reset เพื่อ set ค่าทั้งหมดเป็น 0
- ผู้เล่นสามารถลงเงินและเล่นต่อได้ทันทีหลังจ่ายเงิน ETH เสร็จ

```
function _reset() private {
        numPlayer = 0;
        numReveal = 0;
        numCommit = 0;
        commitTimeP0 = 0;
        commitTimeP1 = 0;
        revealTimeP0 = 0;
        revealTimeP1 = 0;
        reward = 0;
        numInput = 0;
        delete player[0];
        delete player[1];
    }
```
5.เกมที่ซับซ้อนมากขึ้น
- เปลี่ยน choice เป็น 7 แบบโดยให้ 0 = Rock, 1 = Fire , 2 = Scissors , 3 = Sponge , 4 = Paper , 5 = Air , 6 = Water
- เปลี่ยนการหาผู้ชนะเป็นแบบดังนี้
```
if ((p0Choice + 1) % 7 == p1Choice || (p0Choice + 2) % 7 == p1Choice || (p0Choice + 3) % 7 == p1Choice ) {
            // to pay player[0]
            account0.transfer(reward);
        }
else if ((p1Choice + 1) % 7 == p0Choice || (p1Choice + 2) % 7 == p0Choice || (p1Choice + 3) % 7 == p0Choice) {
    // to pay player[1]
    account1.transfer(reward);    
}
else {
    // to split reward
    account0.transfer(reward / 2);
    account1.transfer(reward / 2);
}
```
ตัวอย่างการทดสอบ
- กรณีมีผู้ชนะ
  - กด add player ก่อนให้ครบ 2 คนก่อน
![Screenshot 2024-02-13 190722](https://github.com/Pponghub/RWAPSSF/assets/119305998/9da65617-5edd-4936-944e-614fc916576e)
![Screenshot 2024-02-13 191058](https://github.com/Pponghub/RWAPSSF/assets/119305998/6f2e06c8-450a-47e9-a72c-47d5e05a7570)

  - เมื่อมีผู้เล่นครบ 2 คน จึงเริ่มเลือก choice และส่ง input ได้
![Screenshot 2024-02-13 190808](https://github.com/Pponghub/RWAPSSF/assets/119305998/fb4a138b-8391-491c-86d8-ba5565c2efe0)
    ผู้เล่น 0x5B3... เลือกออกไฟและใส่ salt เป็น 111
    
    ![Screenshot 2024-02-13 190825](https://github.com/Pponghub/RWAPSSF/assets/119305998/b5d0857b-eb2d-4418-aac1-eabdb2e0a381)
    ผู้เล่น 0xAb8... เลือกออกกระดาษและใส่ salt เป็น 222
    
  - เมื่อผู้เล่นทั้ง 2 ส่ง input แล้วต้องมายืนยันโดยส่ง revealPlayer ก่อน เกมจึงจะตัดสินได้
![Screenshot 2024-02-13 191037](https://github.com/Pponghub/RWAPSSF/assets/119305998/cc851b03-65ba-4efa-8c16-e15bdcca5b28)
![Screenshot 2024-02-13 191019](https://github.com/Pponghub/RWAPSSF/assets/119305998/28c0d51f-f2b3-42da-a619-a4989c240f1e)

  - เมื่อส่ง reveal ทั้ง 2 ฝ่ายแล้ว เงิน ETH ที่เป็นรางวัลจะถูกส่งไปหาผู้ชนะ (โดยตานี้ชัยชนะเป็นของ 0x5B3...)
![Screenshot 2024-02-13 191145](https://github.com/Pponghub/RWAPSSF/assets/119305998/4910c66b-ed64-444b-ba0c-2a919df9b6bd)
![Screenshot 2024-02-13 191226](https://github.com/Pponghub/RWAPSSF/assets/119305998/a90c3790-b311-4e93-adef-34f5041c968e)

- กรณีที่เสมอ
  - กด add player ก่อนให้ครบ 2 คนก่อน
![Screenshot 2024-02-13 192509](https://github.com/Pponghub/RWAPSSF/assets/119305998/0ea92c56-2b45-4d3f-aa63-218a50e6089a)

  - เมื่อมีผู้เล่นครบ 2 คน จึงเริ่มเลือก choice และส่ง input ได้
![Screenshot 2024-02-13 192616](https://github.com/Pponghub/RWAPSSF/assets/119305998/97cf4168-64d9-4f8d-9e5d-75ab27110da2)
![Screenshot 2024-02-13 192631](https://github.com/Pponghub/RWAPSSF/assets/119305998/1c1d7abb-eebd-4ff9-aad8-f2f04304ce0c)
    ทั้ง 2 คนเลือก choice เดียวกันคือ น้ำ

  - เมื่อส่ง reveal ทั้ง 2 ฝ่ายแล้ว เงิน ETH ที่เป็นรางวัลจะถูกส่งไปหาผู้ชนะ ซึ่งในกรณีที่เสมอ เงินจะถูกแบ่งครึ่งให้กับผู้เล่นทั้ง 2 คนเท่า ๆ กัน คือ 1 ETH
    ![Screenshot 2024-02-13 192705](https://github.com/Pponghub/RWAPSSF/assets/119305998/48d184c4-2b88-41df-8659-9a0a200eae24)
    ![Screenshot 2024-02-13 192711](https://github.com/Pponghub/RWAPSSF/assets/119305998/18750eff-4966-418b-b839-f544e96310cf)
