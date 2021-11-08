> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.csdn.net](https://blog.csdn.net/chence19871/article/details/26388669)

逆向的时候从 IDA 里抠出来的，对比 MD5 的算法原理，添加上了自己的注释，加深对 MD5 算法的理解。

为了方便，先贴上几个函数：

四个线性函数 (& 是与,| 是或,~ 是非,^ 是异或)

  F(X,Y,Z) = (X & Y) | (( ~ X) & Z)  
  G(X,Y,Z) = (X & Z) | (Y & ( ~ Z))  
  H(X,Y,Z) = X ^ Y ^ Z  
  I(X,Y,Z) = Y ^ (X | ( ~ Z))

Mj 表示消息的第 j 个子分组（从 0 到 15），<<<s 表示循环左移 s 位，则四种操作为：

  FF(a,b,c,d,Mj,s,ti) 表示 a = b + ((a + F(b,c,d) + Mj + ti) <<< s)  
  GG(a,b,c,d,Mj,s,ti) 表示 a = b + ((a + G(b,c,d) + Mj + ti) <<< s)  
  HH(a,b,c,d,Mj,s,ti) 表示 a = b + ((a + H(b,c,d) + Mj + ti) <<< s)  
  II(a,b,c,d,Mj,s,ti) 表示 a = b + ((a + I(b,c,d) + Mj + ti) <<< s)

母函数（文件校验）：

```
int __cdecl sub_4544D0(LPCSTR lpFileName)
{
  unsigned int v1; // ebx@1
  HANDLE v2; // eax@1
  void *v3; // esi@1
  DWORD file_size; // eax@3
  DWORD v6; // edi@5
  int v7; // [sp-14h] [bp-1138h]@11
  const void *v8; // [sp-10h] [bp-1134h]@11
  signed int v9; // [sp-Ch] [bp-1130h]@11
  DWORD lpNumBytesRead; // [sp+8h] [bp-111Ch]@5
  DWORD NumberOfBytesRead; // [sp+Ch] [bp-1118h]@15
  char v12; // [sp+10h] [bp-1114h]@15
  int v13; // [sp+11h] [bp-1113h]@15
  int v14; // [sp+15h] [bp-110Fh]@15
  int v15; // [sp+19h] [bp-110Bh]@15
  int v16; // [sp+1Dh] [bp-1107h]@15
  int v17; // [sp+21h] [bp-1103h]@15
  int v18; // [sp+25h] [bp-10FFh]@15
  int v19; // [sp+29h] [bp-10FBh]@15
  int v20; // [sp+2Dh] [bp-10F7h]@15
  char v21; // [sp+34h] [bp-10F0h]@15
  const CHAR String2; // [sp+44h] [bp-10E0h]@15
  char v23; // [sp+45h] [bp-10DFh]@15
  char v24; // [sp+6Dh] [bp-10B7h]@15
  CHAR v25; // [sp+70h] [bp-10B4h]@15
  char v26; // [sp+71h] [bp-10B3h]@15
  char v27; // [sp+99h] [bp-108Bh]@15
  CHAR String1; // [sp+9Ch] [bp-1088h]@15
  char v29; // [sp+9Dh] [bp-1087h]@15
  char v30; // [sp+C5h] [bp-105Fh]@15
  int Struc_MD5; // [sp+C8h] [bp-105Ch]@5
  char Buffer; // [sp+120h] [bp-1004h]@6
 
  v1 = 0;
  v2 = CreateFileA(lpFileName, 0x80000000u, 1u, 0, 3u, 0, 0);
  v3 = v2;
  if ( v2 == (HANDLE)-1 )
    return 1;
  file_size = GetFileSize(v2, 0);
  if ( file_size < 0x29 )
  {
    CloseHandle(v3);
    return -1;
  }
  v6 = file_size - 0x29;
  InitMD5(&Struc_MD5);//MD5初始化
  lpNumBytesRead = 0;
  if ( v6 <= 0x1000 )
  {
    if ( ReadFile(v3, &Buffer, v6, &lpNumBytesRead, 0) )
    {
      v9 = lpNumBytesRead;
      v8 = &Buffer;
      v7 = (int)&Struc_MD5;
LABEL_14:
      md5_calc(v7, v8, v9);
      goto LABEL_15;
    }
  }
  else
  {
    if ( ReadFile(v3, &Buffer, 0x1000u, &lpNumBytesRead, 0) )
    {
      while ( 1 )
      {
        v1 += lpNumBytesRead;
        md5_calc((int)&Struc_MD5, &Buffer, lpNumBytesRead);// 累加计算MD5
        if ( v1 >= v6 - 0x1000 )
        {
          if ( ReadFile(v3, &Buffer, v6 - v1, &lpNumBytesRead, 0) )
            break;
        }
        if ( !ReadFile(v3, &Buffer, 0x1000u, &lpNumBytesRead, 0) )//循环读取文件数据
          goto LABEL_15;
      }
      v9 = lpNumBytesRead;
      v8 = &Buffer;
      v7 = (int)&Struc_MD5;
      goto LABEL_14;
    }
  }
LABEL_15:
  sub_458D80(&Struc_MD5, &v21);
  v13 = 0;
  v14 = 0;
  v15 = 0;
  v16 = 0;
  v17 = 0;
  v18 = 0;
  v19 = 0;
  v12 = 0;
  v20 = 0;
  sub_454310(&v21, 16, &v12);
  v25 = 0;
  memset(&v26, 0, 0x28u);
  v27 = 0;
  wsprintfA(&v25, "\r\n<!--%s-->", &v12);
  String2 = 0;
  memset(&v23, 0, 0x28u);
  v24 = 0;
  NumberOfBytesRead = 0;
  ReadFile(v3, (LPVOID)&String2, 0x29u, &NumberOfBytesRead, 0);
  CloseHandle(v3);
  String1 = 0;
  memset(&v29, 0, 0x28u);
  v30 = 0;
  lstrcpyA(&String1, &String2);
  return -(lstrcmpA(&v25, &String1) != 0);
}
```

MD5 初始化

```
int __cdecl InitMD5(int a1)
{
  int result; // eax@1
 
  result = a1;
  *(_DWORD *)(a1 + 4) = 0;
  *(_DWORD *)a1 = 0;
  *(_DWORD *)(a1 + 8) = 0x67452301u;
  *(_DWORD *)(a1 + 0xC) = 0xEFCDAB89u;
  *(_DWORD *)(a1 + 0x10) = 0x98BADCFEu;
  *(_DWORD *)(a1 + 0x14) = 0x10325476u;
  return result;
}
```

  
 这个函数是用来计算累加数据的 MD5，比如接收到某一大段数据里的某一小段数据后更新 MD5：

```
int __cdecl md5_calc(int p_struc_md5, const void *data, signed int data_len)
{
  char *pData_t; // esi@1
  int result; // eax@1
  signed int data_len_t; // edi@1
  int v6; // ecx@1
  signed int v7; // ecx@6
  unsigned int v8; // ebp@11
 
  pData_t = (char *)data;
  result = (*(_DWORD *)p_struc_md5 >> 3) & 0x3F;
  data_len_t = data_len;
  v6 = 8 * data_len;
  if ( data_len <= 0 )
    return result;
  *(_DWORD *)p_struc_md5 += v6;
  *(_DWORD *)(p_struc_md5 + 4) += data_len >> 0x1Du;
  if ( *(_DWORD *)p_struc_md5 < (unsigned int)v6 )
    ++*(_DWORD *)(p_struc_md5 + 4);
  if ( result )
  {
    if ( result + data_len <= (signed int)0x40u )
      v7 = data_len;
    else
      v7 = 0x40 - result;
    memcpy((void *)(result + p_struc_md5 + 0x18), data, v7);
    result += v7;
    if ( result < 64 )
      return result;
    pData_t = (char *)data + v7;
    data_len_t = data_len - v7;
    result = MD5OneTurn((char *)(p_struc_md5 + 0x18), p_struc_md5);
  }
  if ( data_len_t >= 64 )
  {
    v8 = (unsigned int)data_len_t >> 6;
    data_len_t += -64 * ((unsigned int)data_len_t >> 6);
    do
    {
      result = MD5OneTurn(pData_t, p_struc_md5);
      pData_t += 64;
      --v8;
    }
    while ( v8 );
  }
  if ( data_len_t )
    memcpy((void *)(p_struc_md5 + 0x18), pData_t, data_len_t);
  return result;
}
```

  
这个函数计算了一轮 MD5：

```
int __usercall MD5OneTurn<eax>(char *data<eax>, int MD5Out)
{
  int a; // edx@1
  int b; // ebx@1
  int c; // ebp@1
  unsigned int v5; // edx@3
  int v6; // ecx@3
  unsigned int v7; // esi@3
  int v8; // edx@3
  unsigned int v9; // edi@3
  int v10; // esi@3
  unsigned int v11; // ebx@3
  int v12; // edi@3
  int v13; // ST10_4@3
  int v14; // edi@3
  unsigned int v15; // edi@3
  unsigned int v16; // edx@3
  int v17; // ecx@3
  unsigned int v18; // esi@3
  int v19; // edx@3
  int v20; // ST14_4@3
  unsigned int v21; // esi@3
  unsigned int v22; // ST10_4@3
  int v23; // ST34_4@3
  int v24; // esi@3
  unsigned int v25; // esi@3
  unsigned int v26; // ecx@3
  int v27; // edi@3
  unsigned int v28; // edx@3
  int v29; // ecx@3
  unsigned int v30; // ST10_4@3
  unsigned int v31; // esi@3
  int v32; // edx@3
  unsigned int v33; // edi@3
  int v34; // esi@3
  signed int v35; // ebx@3
  unsigned int v36; // ecx@3
  int v37; // edi@3
  signed int v38; // ST18_4@3
  unsigned int v39; // ebp@3
  int v40; // ecx@3
  unsigned int v41; // edx@3
  int v42; // ebx@3
  unsigned int v43; // esi@3
  int v44; // edx@3
  unsigned int v45; // edi@3
  int v46; // esi@3
  unsigned int v47; // ecx@3
  int v48; // edi@3
  int v49; // ebp@3
  unsigned int v50; // ebx@3
  int v51; // ST10_4@3
  int v52; // ST24_4@3
  unsigned int v53; // edx@3
  int v54; // ecx@3
  int v55; // edx@3
  int v56; // ecx@3
  unsigned int v57; // esi@3
  int v58; // edx@3
  signed __int64 v59; // qdi@3
  unsigned int v60; // ebx@3
  int v61; // ST10_4@3
  unsigned int v62; // ecx@3
  int v63; // ecx@3
  unsigned int v64; // edx@3
  int v65; // ecx@3
  int v66; // edx@3
  unsigned int v67; // ebx@3
  int v68; // ST10_4@3
  int v69; // ST18_4@3
  unsigned int v70; // ecx@3
  int v71; // ecx@3
  int v72; // ecx@3
  int v73; // edx@3
  int v74; // ebx@3
  int v75; // ebx@3
  int v76; // ST44_4@3
  unsigned int v77; // ecx@3
  int v78; // ST20_4@3
  int v79; // ST30_4@3
  int v80; // ecx@3
  int v81; // edx@3
  int v82; // ecx@3
  int v83; // ST40_4@3
  unsigned int v84; // edx@3
  int v85; // ebx@3
  unsigned int v86; // edx@3
  int v87; // ST2C_4@3
  int v88; // edx@3
  int v89; // ST3C_4@3
  int v90; // ecx@3
  unsigned int v91; // ebx@3
  int v92; // ST1C_4@3
  int v93; // ST28_4@3
  unsigned int v94; // ebx@3
  int v95; // ST38_4@3
  unsigned int v96; // edx@3
  int v97; // ecx@3
  int v98; // ST48_4@3
  int v99; // ebp@3
  int v100; // eax@3
  unsigned int v101; // ebx@3
  unsigned int v102; // ebx@3
  int v103; // edx@3
  signed __int64 v104; // qcx@3
  int v105; // edx@3
  int v106; // edx@3
  int v107; // eax@3
  unsigned int v108; // edx@3
  unsigned int v109; // eax@3
  unsigned int v110; // edx@3
  int result; // eax@3
  int d; // [sp+10h] [bp-7Ch]@1
  char v113; // [sp+4Ch] [bp-40h]@2
 
  a = *(_DWORD *)(MD5Out + 8);                  // 四个幻数
  b = *(_DWORD *)(MD5Out + 0xC);
  c = *(_DWORD *)(MD5Out + 0x10);
  d = *(_DWORD *)(MD5Out + 0x14);
  if ( (unsigned __int8)data & 3 )              // 地址对齐
  {
    memcpy(&v113, data, 0x40u);
    data = &v113;
  }                                             // ======================第一轮开始==================
                                                // 
  v5 = *(_DWORD *)data + (b & c | d & ~b) + a - 0x28955B88;
  v6 = b + ((v5 << 7) | (v5 >> 25));            // 循环左移7位   a=FF(a,b,c,d,M0,7,0xd76aa478)
                                                // 
  v7 = *((_DWORD *)data + 1) + (v6 & b | c & ~v6) + d - 0x173848AA;
  v8 = v6 + ((v7 << 12) | (v7 >> 20));          // d=FF(d,a,b,c,M1,12,0xe8c7b756)
                                                // 
  v9 = *((_DWORD *)data + 2) + (v6 & v8 | b & ~(v6 + ((v7 << 12) | (v7 >> 20)))) + c + 0x242070DB;
  v10 = v8 + ((v9 << 17) | (v9 >> 15));         // c=FF(c,d,a,b,M2,17,0x242070db)
                                                // 
  v11 = *((_DWORD *)data + 3) + (v10 & v8 | v6 & ~(v8 + ((v9 << 17) | (v9 >> 15)))) + b - 0x3E423112;
  v12 = v10 + ((v11 >> 10) | (v11 << 22));      // b=FF(b,c,d,a,M3,22,0xc1bdceee)
                                                // 
  v13 = v12;
  v14 = *((_DWORD *)data + 4) + (v12 & v10 | v8 & ~v12);
  v15 = v13 + (((v14 + v6 - 0xA83F051) << 7) | ((v14 + v6 - 0xA83F051u) >> 25));// a=FF(a,b,c,d,M4,7,0xf57c0faf)
                                                // 
  v16 = *((_DWORD *)data + 5) + (v15 & v13 | v10 & ~v15) + v8 + 0x4787C62A;
  v17 = v15 + ((v16 << 12) | (v16 >> 20));      // d=FF(d,a,b,c,M5,12,0x4787c62a)
                                                // 
  v18 = *((_DWORD *)data + 6) + (v15 & v17 | v13 & ~v17) + v10 - 0x57CFB9ED;
  v19 = v17 + ((v18 << 17) | (v18 >> 15));      // c=FF(c,d,a,b,M6,17,0xa8304613)
                                                // 
  v20 = *((_DWORD *)data + 7);
  v21 = v19
      + (((v20 + (v19 & v17 | v15 & ~v19) + v13 - 0x2B96AFF) >> 10) | ((v20 + (v19 & v17 | v15 & ~v19) + v13 - 0x2B96AFF) << 22));// b=FF(b,c,d,a,M7,22,0xfd469501)
                                                // 
  v22 = v21;
  v23 = *((_DWORD *)data + 8);
  v24 = v23 + (v21 & v19 | v17 & ~v21);
  v25 = v22 + (((v24 + v15 + 0x698098D8) << 7) | ((v24 + v15 + 0x698098D8) >> 25));
  v26 = *((_DWORD *)data + 9) + (v25 & v22 | v19 & ~v25) + v17 - 0x74BB0851;
  v27 = v25 + ((v26 << 12) | (v26 >> 20));
  v28 = *((_DWORD *)data + 10) + (v25 & v27 | v22 & ~(v25 + ((v26 << 12) | (v26 >> 20)))) + v19 - 42063;
  v29 = v27 + ((v28 << 17) | (v28 >> 15));
  v30 = v29
      + (((*((_DWORD *)data + 11) + (v29 & v27 | v25 & ~(v27 + ((v28 << 17) | (v28 >> 15)))) + v22 - 0x76A32842) >> 10) | ((*((_DWORD *)data + 11) + (v29 & v27 | v25 & ~(v27 + ((v28 << 17) | (v28 >> 15)))) + v22 - 0x76A32842) << 22));
  v31 = *((_DWORD *)data + 12) + (v30 & v29 | v27 & ~v30) + v25 + 0x6B901122;
  v32 = v30 + ((v31 << 7) | (v31 >> 25));
  v33 = *((_DWORD *)data + 13) + (v32 & v30 | v29 & ~v32) + v27 - 0x2678E6D;
  v34 = v32 + ((v33 << 12) | (v33 >> 20));
  v35 = ~(v32 + ((v33 << 12) | (v33 >> 20)));
  v36 = *((_DWORD *)data + 14) + (v32 & v34 | v30 & ~(v32 + ((v33 << 12) | (v33 >> 20)))) + v29 - 0x5986BC72;
  v37 = v34 + ((v36 << 17) | (v36 >> 15));
  v38 = ~(v34 + ((v36 << 17) | (v36 >> 15)));
  v39 = *((_DWORD *)data + 15) + (v37 & v34 | v32 & v38) + v30 + 0x49B40821;
  v40 = v37 + ((v39 >> 10) | (v39 << 22));      // b=FF(b,c,d,a,M15,22,0x49b40821)
                                                // 
                                                // ============================第二轮========================
  v41 = *((_DWORD *)data + 1) + (v40 & v34 | v37 & v35) + v32 - 0x9E1DA9E;
  v42 = v40 + (32 * v41 | (v41 >> 27));         // a=GG(a,b,c,d,M1,5,0xf61e2562)
                                                // 
  v43 = *((_DWORD *)data + 6) + (v42 & v37 | v40 & v38) + v34 - 0x3FBF4CC0;
  v44 = v42 + ((v43 << 9) | (v43 >> 23));       // d=GG(d,a,b,c,M6,9,0xc040b340)
                                                // 
  v45 = *((_DWORD *)data + 11) + (v40 & v44 | v42 & ~(v37 + ((v39 >> 10) | (v39 << 22)))) + v37 + 0x265E5A51;
  v46 = v44 + ((v45 << 14) | (v45 >> 18));      // c=GG(c,d,a,b,M11,14,0x265e5a51)
                                                // 
  v47 = *(_DWORD *)data + (v42 & v46 | v44 & ~v42) + v40 - 0x16493856;
  v48 = v46 + ((v47 >> 12) | (v47 << 20));      // b=GG(b,c,d,a,M0,20,0xe9b6c7aa)
                                                // 
  v49 = *((_DWORD *)data + 5);
  v50 = v49 + (v48 & v44 | v46 & ~v44) + v42 - 0x29D0EFA3;
  v51 = v48 + (32 * v50 | (v50 >> 27));         // a=GG(a,b,c,d,M5,5,0xd62f105d)
                                                // 
  v52 = *((_DWORD *)data + 5);
  v53 = *((_DWORD *)data + 10) + (v51 & v46 | v48 & ~v46) + v44 + 0x2441453;
  v54 = (v53 << 9) | (v53 >> 23);
  v55 = v48 + (32 * v50 | (v50 >> 27));
  v56 = v55 + v54;                              // d=GG(d,a,b,c,M10,9,0x02441453)
                                                // 
  v57 = *((_DWORD *)data + 15) + (v48 & v56 | v55 & ~v48) + v46 - 0x275E197F;
  v58 = v56 + ((v57 << 14) | (v57 >> 18));      // c=GG(c,d,a,b,M15,14,0xd8a1e681)
                                                // 
  LODWORD(v59) = *((_DWORD *)data + 4) + (v51 & v58 | v56 & ~(v48 + (32 * v50 | (v50 >> 27)))) + v48 - 0x182C0438;
  HIDWORD(v59) = v59;
  HIDWORD(v59) = v58 + (unsigned __int64)(v59 >> 12);// 此处用了_int64,高低两个DWORD相等，再移位，再取低DWORD，等价于一个DWORD的循环移位。
                                                // b=GG(b,c,d,a,M4,20,0xe7d3fbc8)
                                                // 
  v60 = *((_DWORD *)data + 9) + (HIDWORD(v59) & v56 | v58 & ~v56) + v51 + 0x21E1CDE6;
  v61 = HIDWORD(v59) + (32 * v60 | (v60 >> 27));// a=GG(a,b,c,d,M9,5,0x21e1cde6)
                                                // 
  v62 = *((_DWORD *)data + 14) + (v61 & v58 | HIDWORD(v59) & ~v58) + v56 - 0x3CC8F82A;
  LODWORD(v59) = (v62 << 9) | (v62 >> 23);
  v63 = HIDWORD(v59) + (32 * v60 | (v60 >> 27));
  LODWORD(v59) = v63 + v59;                     // d=GG(d,a,b,c,M14,9,0xc33707d6)
                                                // 
  v64 = *((_DWORD *)data + 3) + (HIDWORD(v59) & v59 | v63 & ~HIDWORD(v59)) + v58 - 0xB2AF279;
  v65 = v59 + ((v64 << 14) | (v64 >> 18));      // c=GG(c,d,a,b,M3,14,0xf4d50d87)
                                                // 
  HIDWORD(v59) += *((_DWORD *)data + 8) + (v61 & v65 | v59 & ~(HIDWORD(v59) + (32 * v60 | (v60 >> 27)))) + 0x455A14ED;
  v66 = v65 + ((HIDWORD(v59) >> 12) | (HIDWORD(v59) << 20));// b=GG(b,c,d,a,M8,20,0x455a14ed)
                                                // 
  v67 = *((_DWORD *)data + 13) + (v66 & v59 | v65 & ~(_DWORD)v59) + v61 - 0x561C16FB;
  v68 = v66 + (32 * v67 | (v67 >> 27));         // a=GG(a,b,c,d,M13,5,0xa9e3e905)
                                                // 
  LODWORD(v59) = *((_DWORD *)data + 2) + (v68 & v65 | v66 & ~v65) + v59 - 0x3105C08;
  v69 = *((_DWORD *)data + 2);
  HIDWORD(v59) = v68 + (((_DWORD)v59 << 9) | ((unsigned int)v59 >> 23));// d=GG(d,a,b,c,M2,9,0xfcefa3f8)
                                                // 
  v70 = *((_DWORD *)data + 7)
      + (v66 & (v68 + (((_DWORD)v59 << 9) | ((unsigned int)v59 >> 23))) | v68 & ~v66)// 这里F5冗余了，相当于HIDWORD(v59)
      + v65
      + 0x676F02D9;
  LODWORD(v59) = HIDWORD(v59) + ((v70 << 14) | (v70 >> 18));// c=GG(c,d,a,b,M7,14,0x676f02d9)
                                                // 
  v71 = *((_DWORD *)data + 12) + (v68 & v59 | HIDWORD(v59) & ~(v66 + (32 * v67 | (v67 >> 27))));
  v72 = v59 + ((signed __int64)__PAIR__(v71 + v66 - 0x72D5B376u, v71 + v66 - 0x72D5B376u) >> 12);// b=GG(b,c,d,a,M12,20,0x8d2a4c8a)
                                                // 
                                                // ======================第三轮======================
  v73 = v72
      + (16 * (v49 + (v72 ^ v59 ^ HIDWORD(v59)) + v66 + (32 * v67 | (v67 >> 27)) - 0x5C6BE) | ((v49
                                                                                              + (v72 ^ (unsigned int)v59 ^ HIDWORD(v59))
                                                                                              + v66
                                                                                              + (32 * v67 | (v67 >> 27))
                                                                                              - 0x5C6BE) >> 28));// a=HH(a,b,c,d,M5,4,0xfffa3942)
                                                // 
  HIDWORD(v59) = *((_DWORD *)data + 8) + (v73 ^ v72 ^ v59) + HIDWORD(v59) - 0x788E097F;
  v74 = (HIDWORD(v59) << 11) | (HIDWORD(v59) >> 21);
  HIDWORD(v59) = *((_DWORD *)data + 11);
  v75 = v73 + v74;                              // d=HH(d,a,b,c,M8,11,0x8771f681)
                                                // 
  v76 = HIDWORD(v59);
  HIDWORD(v59) += v59 + (v73 ^ v72 ^ v75) + 0x6D9D6122;
  LODWORD(v59) = HIDWORD(v59);
  LODWORD(v59) = v75 + (unsigned __int64)(v59 >> 16);// c=HH(c,d,a,b,M11,16,0x6d9d6122)
                                                // 
  v77 = *((_DWORD *)data + 14) + (v73 ^ v59 ^ v75) + v72 - 0x21AC7F4;
  HIDWORD(v59) = v59 + ((v77 >> 9) | (v77 << 23));// b=HH(b,c,d,a,M14,23,0xfde5380c)
                                                // 
  v78 = *((_DWORD *)data + 14);
  v79 = *((_DWORD *)data + 1);
  v80 = 16 * (v79 + (HIDWORD(v59) ^ v59 ^ v75) + v73 - 0x5B4115BC) | ((v79
                                                                     + (HIDWORD(v59) ^ (unsigned int)v59 ^ v75)
                                                                     + v73
                                                                     - 0x5B4115BC) >> 28);
  v81 = *((_DWORD *)data + 4);
  v82 = HIDWORD(v59) + v80;                     // a=HH(a,b,c,d,M1,4,0xa4beea44)
                                                // 
  v83 = v81;
  v84 = v75 + v81 + (v82 ^ HIDWORD(v59) ^ v59) + 0x4BDECFA9;
  v85 = v82 + ((v84 << 11) | (v84 >> 21));
  v86 = *((_DWORD *)data + 7) + (v82 ^ HIDWORD(v59) ^ (v82 + ((v84 << 11) | (v84 >> 21)))) + v59 - 0x944B4A0;
  LODWORD(v59) = v85 + ((v86 << 16) | (v86 >> 16));
  HIDWORD(v59) = HIDWORD(v59) + *((_DWORD *)data + 10) + (v82 ^ v59 ^ v85) - 0x41404390;
  v87 = *((_DWORD *)data + 10);
  v88 = v59 + ((HIDWORD(v59) >> 9) | (HIDWORD(v59) << 23));
  v89 = *((_DWORD *)data + 13);
  HIDWORD(v59) = v89 + (v88 ^ v59 ^ v85) + v82 + 0x289B7EC6;
  v90 = v88 + (16 * HIDWORD(v59) | (HIDWORD(v59) >> 28));
  v91 = v85 + *(_DWORD *)data + (v90 ^ v88 ^ v59) - 0x155ED806;
  v92 = *(_DWORD *)data;
  HIDWORD(v59) = v90 + ((v91 << 11) | (v91 >> 21));
  LODWORD(v59) = v59 + *((_DWORD *)data + 3) + (v90 ^ v88 ^ HIDWORD(v59)) - 0x2B10CF7B;
  v93 = *((_DWORD *)data + 3);
  v94 = HIDWORD(v59) + (((_DWORD)v59 << 16) | ((unsigned int)v59 >> 16));
  LODWORD(v59) = *((_DWORD *)data + 6);
  v95 = v59;
  LODWORD(v59) = v88 + v59 + (v90 ^ v94 ^ HIDWORD(v59)) + 0x4881D05;
  v96 = v94 + (((unsigned int)v59 >> 9) | ((_DWORD)v59 << 23));
  LODWORD(v59) = *((_DWORD *)data + 9) + (v96 ^ v94 ^ HIDWORD(v59)) + v90 - 0x262B2FC7;
  v97 = v96 + (16 * v59 | ((unsigned int)v59 >> 28));
  v98 = *((_DWORD *)data + 9);
  v99 = *((_DWORD *)data + 12);
  v100 = *((_DWORD *)data + 15);
  LODWORD(v59) = v99 + (v97 ^ v96 ^ v94) + HIDWORD(v59) - 0x1924661B;
  HIDWORD(v59) = v97 + (((_DWORD)v59 << 11) | ((unsigned int)v59 >> 21));
  v101 = v100 + (v97 ^ v96 ^ (v97 + (((_DWORD)v59 << 11) | ((unsigned int)v59 >> 21)))) + v94 + 0x1FA27CF8;
  LODWORD(v59) = HIDWORD(v59) + ((v101 << 16) | (v101 >> 16));// c=HH(c,d,a,b,M15,16,0x1fa27cf8)
                                                // 
  v102 = v69 + (v97 ^ v59 ^ HIDWORD(v59)) + v96 - 0x3B53A99B;
  v103 = v59 + ((v102 >> 9) | (v102 << 23));    // b=HH(b,c,d,a,M2,23,0xc4ac5665)
                                                // 
                                                // =========================第四轮=======================
  LODWORD(v104) = v92 + (v59 ^ (v103 | ~HIDWORD(v59))) + v97 - 0xBD6DDBC;
  HIDWORD(v104) = v104;
  LODWORD(v104) = v103 + (unsigned __int64)(v104 >> 26);// a=II(a,b,c,d,M0,6,0xf4292244)
                                                // 
  HIDWORD(v104) = v20 + (v103 ^ (v104 | ~(_DWORD)v59)) + HIDWORD(v59) + 0x432AFF97;
  HIDWORD(v59) = v104 + ((HIDWORD(v104) << 10) | (HIDWORD(v104) >> 22));
  HIDWORD(v104) = v78 + (v104 ^ (HIDWORD(v59) | ~v103)) + (_DWORD)v59 - 0x546BDC59;
  LODWORD(v59) = HIDWORD(v59) + ((HIDWORD(v104) << 15) | (HIDWORD(v104) >> 17));
  HIDWORD(v104) = v52 + (HIDWORD(v59) ^ (v59 | ~(_DWORD)v104)) + v103 - 0x36C5FC7;
  v105 = v59 + ((HIDWORD(v104) >> 11) | (HIDWORD(v104) << 21));
  LODWORD(v104) = v99 + (v59 ^ (v105 | ~HIDWORD(v59))) + (_DWORD)v104 + 0x655B59C3;
  HIDWORD(v104) = v104;
  LODWORD(v104) = v105 + (unsigned __int64)(v104 >> 26);
  HIDWORD(v59) = v104
               + (((v93 + (v105 ^ ((unsigned int)v104 | ~(_DWORD)v59)) + HIDWORD(v59) - 0x70F3336E) << 10) | ((v93 + (v105 ^ ((unsigned int)v104 | ~(_DWORD)v59)) + HIDWORD(v59) - 0x70F3336E) >> 22));
  LODWORD(v59) = HIDWORD(v59)
               + (((v87 + ((unsigned int)v104 ^ (HIDWORD(v59) | ~v105)) + (_DWORD)v59 - 0x100B83) << 15) | ((v87 + ((unsigned int)v104 ^ (HIDWORD(v59) | ~v105)) + (unsigned int)v59 - 0x100B83) >> 17));
  v106 = v59
       + (((v79 + (HIDWORD(v59) ^ ((unsigned int)v59 | ~(_DWORD)v104)) + v105 - 0x7A7BA22F) >> 11) | ((v79 + (HIDWORD(v59) ^ ((unsigned int)v59 | ~(_DWORD)v104)) + v105 - 0x7A7BA22F) << 21));
  LODWORD(v104) = v106
                + (((v23 + ((unsigned int)v59 ^ (v106 | ~HIDWORD(v59))) + (_DWORD)v104 + 0x6FA87E4F) << 6) | ((v23 + ((unsigned int)v59 ^ (v106 | ~HIDWORD(v59))) + (unsigned int)v104 + 0x6FA87E4F) >> 26));
  HIDWORD(v59) = v100 + (v106 ^ (v104 | ~(_DWORD)v59)) + HIDWORD(v59) - 0x1D31920;
  v107 = v104 + ((HIDWORD(v59) << 10) | (HIDWORD(v59) >> 22));
  LODWORD(v59) = v95 + (v104 ^ (v107 | ~v106)) + (_DWORD)v59 - 1560198380;
  HIDWORD(v59) = v107 + (((_DWORD)v59 << 15) | ((unsigned int)v59 >> 17));
  LODWORD(v59) = v89 + (v107 ^ (HIDWORD(v59) | ~(_DWORD)v104)) + v106 + 0x4E0811A1;
  v108 = HIDWORD(v59) + (((unsigned int)v59 >> 11) | ((_DWORD)v59 << 21));
  LODWORD(v104) = v108
                + (((v83 + (HIDWORD(v59) ^ (v108 | ~v107)) + (_DWORD)v104 - 0x8AC817E) << 6) | ((v83
                                                                                               + (HIDWORD(v59) ^ (v108 | ~v107))
                                                                                               + (unsigned int)v104
                                                                                               - 0x8AC817E) >> 26));// a=II(a,b,c,d,M4,6,0xf7537e82)
                                                // 
  v109 = v76 + (v108 ^ (v104 | ~HIDWORD(v59))) + v107 - 0x42C50DCB;
  LODWORD(v59) = v104 + ((v109 << 10) | (v109 >> 22));// d=II(d,a,b,c,M11,10,0xbd3af235)
                                                // 
  HIDWORD(v59) += v69 + (v104 ^ (v59 | ~v108)) + 0x2AD7D2BB;
  HIDWORD(v104) = v59 + ((HIDWORD(v59) << 15) | (HIDWORD(v59) >> 17));// c=II(c,d,a,b,M2,15,0x2ad7d2bb)
                                                // 
  v110 = v98 + (v59 ^ (HIDWORD(v104) | ~(_DWORD)v104)) + v108 - 0x14792C6F;
  result = MD5Out;
  *(_DWORD *)(MD5Out + 8) += v104;              // 幻数累加    a
  *(_DWORD *)(result + 12) += HIDWORD(v104) + ((v110 >> 11) | (v110 << 21));// b=II(b,c,d,a,M9,21,0xeb86d391)
                                                // 
  *(_DWORD *)(result + 16) += HIDWORD(v104);    // c
  *(_DWORD *)(MD5Out + 20) += v59;              // d
  return result;
}
```

  
  
最后将 MD5 值保存在 MD5 结构偏移为 8 的位置里，共 16 个字节。