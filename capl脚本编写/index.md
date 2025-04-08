# Capl脚本编写


# Capl 脚本 编写 过程记录



## 1、CMAC 算法编写记录

需要使用CANOE自带的算法库，导入方式为右键创建的节点，然后点Configuration-Components-Add

加载此模块需要在本地搜索下SecMgrCANoeClient.vmodule



正常情况会在函数栏中，看到此安全库，如果没有点下环境导入，CAMC函数为SecurityLocalGenerateCMAC。

```
/*@!Encoding:936*/
//网关2701 认证测试 cmac算法
variables
{
    message 0x70E diagRequestMsg;  // 诊断请求消息
    message 0x702 authRequestMsg;  // 认证请求消息
    message 0x70E flowControl;
    byte key1[16] = { 0x00, 0x00, 0x00, 0x00, 
    0x00, 0x00, 0x00, 0x00, 
    0x00, 0x00, 0x00, 0x00, 
    0x00, 0x00, 0x00, 0x00  }; // AES-128 密钥

    byte seed[16];  // ECU 返回的 Seed
    byte tempSeed[18]; // 用于存储完整的 18 字节 Seed 响应数据
    dword seedLength = 32;

    byte cmac[16]; // CMAC 计算结果
    dword cmacLength = 16;

    int sessionActive = 0;      // 标记是否已进入扩展会话
    int waitingForMultiFrame = 0;  // 标记是否正在接收多帧数据
    int receivedBytes = 0;         // 记录已接收的字节数
    int i;
    long result;
    byte fullSeed[32];  // 目标数组 (16 + 1 + 15 = 32 字节)
}

/* 按 'W' 触发完整安全认证流程 */
on key 'w'
{
    // 发送 1003 进入扩展会话
    diagRequestMsg.dlc = 8;  
    diagRequestMsg.byte(0) = 0x02;
    diagRequestMsg.byte(1) = 0x10;
    diagRequestMsg.byte(2) = 0x03;
    
    output(diagRequestMsg);
    write("?? 发送 1003 进入扩展会话...");
}

/* 监听 ECU 响应 */
on message CAN1.*
{
    /* 处理 ECU 对 1003 的响应 */
    if (this.id == 0x78E && this.byte(1) == 0x50 && this.byte(2) == 0x03)
    {
        write("? 进入扩展会话成功! 继续发送 2701 请求 Seed...");

        sessionActive = 1; // 标记已进入扩展会话

        // 发送 2701 请求 Seed
        diagRequestMsg.dlc = 8;
        diagRequestMsg.byte(0) = 0x02;
        diagRequestMsg.byte(1) = 0x27;
        diagRequestMsg.byte(2) = 0x01;

        output(diagRequestMsg);
        write("?? 发送 2701 请求 Seed...");
    }

    /* 处理 ECU 返回的 Seed (首帧) */
    if (sessionActive == 1 && this.id == 0x78E && this.byte(2) == 0x67)
    {
        waitingForMultiFrame = 1;  // 开始等待多帧数据
        receivedBytes = this.byte(1); // 提取数据长度 (0x12 = 18 字节)

        write("? ECU 发送多帧响应，总数据长度: %d 字节", receivedBytes);

        // 存储首帧数据 (从 byte(2) 开始)
        for (i = 0; i < 6; i++)
        {
            tempSeed[i] = this.byte(i + 2);
        }

        // 发送流控帧 (FC) [30 00 00]
        
        // 发送到 ECU
        flowControl.dlc = 8;
        flowControl.byte(0) = 0x30; // FC 流控帧
        flowControl.byte(1) = 0x00; // Block Size = 8
        flowControl.byte(2) = 0x00; // Separation Time = 20ms
        flowControl.byte(3) = 0x00;
        flowControl.byte(4) = 0x00;
        flowControl.byte(5) = 0x00;
        flowControl.byte(6) = 0x00;
        flowControl.byte(7) = 0x00;

        output(flowControl);
        write("?? 发送流控帧 (FC) -> [30 00 00 00 00 00 00 00]");
    }

    /* 处理 ECU 发送的后续帧 (CF) */
    if (waitingForMultiFrame == 1 && this.id == 0x78e && (this.byte(0) & 0xF0) == 0x20)
    {
        int frameIndex;  // 先声明变量
        int offset;
        frameIndex = (this.byte(0) & 0x0F); // 运行时获取帧序号
       
        offset = 6 + (frameIndex - 1) * 7; // 计算存储偏移量

        write("?? 收到连续帧 #%d, 数据:", frameIndex);

        for (i = 0; i < 7 && (offset + i) < receivedBytes; i++)
        {
            tempSeed[offset + i] = this.byte(i + 1);
            write("0x%02X ", tempSeed[offset + i]);
        }

        // 如果已接收所有数据
        if (offset + i >= receivedBytes)
        {
            waitingForMultiFrame = 0; // 结束多帧接收
            write("? Seed 接收完成, 开始计算 CMAC...");

            // 复制 16 字节 Seed (从 tempSeed[2] 开始)
            for (i = 0; i < 16; i++)
            {
                seed[i] = tempSeed[i + 2];
            }

            write("?? 最终 Seed 数据:");
            write("seed: %02X %02X %02X %02X %02X %02X %02X %02X", seed[0], seed[1], seed[2], seed[3], seed[4], seed[5], seed[6], seed[7]);
            write("      %02X %02X %02X %02X %02X %02X %02X %02X", seed[8], seed[9], seed[10], seed[11], seed[12], seed[13], seed[14], seed[15]);
            // 复制 seed (前 16 字节)
for (i = 0; i < 16; i++) {
    fullSeed[i] = seed[i];
}

// 拼接 0x01
fullSeed[16] = 0x01;

// 拼接 15 个 0x00
for (i = 17; i < 32; i++) {
    fullSeed[i] = 0x00;
}
write("完整 fullSeed 数据: ");
write("fullSeed: %02X %02X %02X %02X %02X %02X %02X %02X", fullSeed[0], fullSeed[1], fullSeed[2], fullSeed[3], fullSeed[4], fullSeed[5], fullSeed[6], fullSeed[7]);
            write("      %02X %02X %02X %02X %02X %02X %02X %02X", fullSeed[8], fullSeed[9], fullSeed[10], fullSeed[11], fullSeed[12], fullSeed[13], fullSeed[14], fullSeed[15]);
write("      %02X %02X %02X %02X %02X %02X %02X %02X", fullSeed[16], fullSeed[17], fullSeed[18], fullSeed[19], fullSeed[20], fullSeed[21], fullSeed[22], fullSeed[23]);
write("      %02X %02X %02X %02X %02X %02X %02X %02X", fullSeed[24], fullSeed[25], fullSeed[26], fullSeed[27], fullSeed[28], fullSeed[29], fullSeed[30], fullSeed[31]);
            // 计算 16 字节 CMAC
            result = SecurityLocalGenerateCMAC(key1, 16, fullSeed, 32, cmac, cmacLength);

            if (result != 1)
            {
                write("? CMAC 计算失败, 错误代码: %d", result);
                return;
            }
            write("CMAC: %02X %02X %02X %02X %02X %02X %02X %02X", cmac[0], cmac[1], cmac[2], cmac[3], cmac[4], cmac[5], cmac[6], cmac[7]);
            write("      %02X %02X %02X %02X %02X %02X %02X %02X", cmac[8], cmac[9], cmac[10], cmac[11], cmac[12], cmac[13], cmac[14], cmac[15]);
            write("? CMAC 计算成功, 发送 2702 进行认证...");

            // 发送 2702 + 16 字节 CMAC
            authRequestMsg.id = 0x70E;
            authRequestMsg.dlc = 8;
            authRequestMsg.byte(0) = 0x10;
            authRequestMsg.byte(1) = 0x12;
            authRequestMsg.byte(2) = 0x27;
            authRequestMsg.byte(3) = 0x02;

            for (i = 0; i < 4; i++)//cmac 前4位 第一帧
            {
                authRequestMsg.byte(i + 4) = cmac[i];
            }

            output(authRequestMsg);//发送第一帧
            write("?? 发送 2702 认证请求第一帧...");
            
            
        }
        
     }
     
     if (sessionActive == 1 && this.id == 0x78e && this.byte(0) == 0x30 )
    {
        write("? 发送第二帧!");
        authRequestMsg.dlc = 8;
        authRequestMsg.byte(0) = 0x21;
        for (i = 0; i < 7; i++)//cmac 第二帧
            {
                authRequestMsg.byte(i + 1) = cmac[i+4];// 4 5 6 7 8 9 10 
            }
          output(authRequestMsg);//发送第二帧
          write("? 发送第三帧!");
          authRequestMsg.dlc = 8;
          authRequestMsg.byte(0) = 0x22;
            for (i = 0; i < 5; i++)//cmac  第三帧
            {
                authRequestMsg.byte(i + 1) = cmac[i+11];// 11 12 13 14 15
            }
            
            for (i = 6; i < 8; i++)//cmac  第三帧尾部填充CC 两位
            {
                authRequestMsg.byte(i) = 0x00;
            }
            
        output(authRequestMsg);//发送第三帧
        
    }

    /* 监听 2702 认证结果 */
    if (sessionActive == 1 && this.id == 0x78e && this.byte(1) == 0x67 && this.byte(2) == 0x02)
    {
        write("? 认证成功, ECU 通过安全访问!");
    }
    else if (sessionActive == 1 && this.id == 0x78e && this.byte(1) == 0x7F)
    {
        write("? 认证失败, 错误码: 0x%02x", this.byte(3));
    }
}
```



## 2、异或算法编写记录

```
/*@!Encoding:936*/
variables
{
    message 0x702 diagRequestMsg;  // 诊断请求消息
   

   

    byte seed[4];  // ECU 返回的 Seed
    byte Cal[4];   // 计算数组
    byte key1[4] = {0x00, 0x00, 0x00, 0x00}; // 密钥数组
    byte Xor[4] = {0x22, 0x22, 0x22, 0x22}; // 修改为对应部件的异或数组

    int sessionActive = 0;      // 标记是否已进入扩展会话
    
    int i;
}

/* 按 'W' 触发完整安全认证流程 */
on key 'e'
{
    // 发送 1003 进入扩展会话
    diagRequestMsg.dlc = 8;  
    diagRequestMsg.byte(0) = 0x02;
    diagRequestMsg.byte(1) = 0x10;
    diagRequestMsg.byte(2) = 0x03;
    
    output(diagRequestMsg);
    write("?? 发送 1003 进入扩展会话...");
}

/* 监听 ECU 响应 */
on message CAN1.*
{
    /* 处理 ECU 对 1003 的响应 */
    if (this.id == 0x782 && this.byte(1) == 0x50 && this.byte(2) == 0x03)
    {
       sessionActive = 1; // 标记已进入扩展会话
        diagRequestMsg.dlc = 8;
        diagRequestMsg.byte(0) = 0x02;
        diagRequestMsg.byte(1) = 0x27;
        diagRequestMsg.byte(2) = 0x01;

        output(diagRequestMsg);
        write("?? 发送 2701 请求 Seed...");
    }
      
      if (sessionActive == 1 && this.id == 0x782 && this.byte(1) == 0x67&&this.byte(2)==0x01){
        write("? 收到 ECU Seed, 开始计算 Key...");
        for ( i = 0; i < 4; i++)
        {
            seed[i] = this.byte(i + 3);
        }

        // 打印 Seed
       write("?? Seed 数据: 0x%02X 0x%02X 0x%02X 0x%02X", seed[0], seed[1], seed[2], seed[3]);

        
        for ( i = 0; i < 4; i++)
        {
            Cal[i] = seed[i] ^ Xor[i];
        }

        // **Step 2: 计算 Key**
        key1[0] = ((Cal[0] & 0x0F) << 4) | (Cal[1] & 0xF0);
        key1[1] = ((Cal[1] & 0x0F) << 4) | ((Cal[2] & 0xF0) >> 4);
        key1[2] = (Cal[2] & 0xF0) | ((Cal[3] & 0xF0) >> 4);
        key1[3] = ((Cal[3] & 0x0F) << 4) | (Cal[0] & 0x0F);

        

        // **打印 Key 及最终值**
        write("?? 计算出的 Key: 0x%02X 0x%02X 0x%02X 0x%02X", key1[0], key1[1], key1[2], key1[3]);
       
       
        // 发送 2702 认证请求
        diagRequestMsg.dlc = 8;
        diagRequestMsg.byte(0) = 0x06;
        diagRequestMsg.byte(1) = 0x27;
        diagRequestMsg.byte(2) = 0x02;
        diagRequestMsg.byte(3) = key1[0];
        diagRequestMsg.byte(4) = key1[1];
        diagRequestMsg.byte(5)= key1[2];
        diagRequestMsg.byte(6) = key1[3];
        diagRequestMsg.byte(7) = 0x00;
        output(diagRequestMsg);
        write("?? 发送 2702 认证请求...");
    }

    

    /* 监听 2702 认证结果 */
    if (sessionActive == 1 && this.id == 0x782 && this.byte(1) == 0x67 && this.byte(2) == 0x02)
    {
        write("? 认证成功, ECU 通过安全访问!");
    }
    else if (sessionActive == 1 && this.id == 0x782 && this.byte(1) == 0x7F)
    {
        write("? 认证失败, 错误码: 0x%02x", this.byte(3));
    }

}
```


