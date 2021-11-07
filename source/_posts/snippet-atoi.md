title: Atoi&Itoa函数
category: snippet 
date: 2018-01-28
tags: [atoi,string2ll,snippet]
toc: false
comments: true
---

- atoi函数实现

```bash
/*************************************************************************
 > File Name: Atoi.c
 > Author: qinchao
 > Mail: 1187620726@qq.com
 > Created Date:2018-01-28 Time:04:36:18.
 ************************************************************************/

// 32bits无符号整数表示范围[0, 2^32-1]
#define UINTEGER32_MAX (4294967295)
// 32bits有符号整数表示范围为[-2^31, 2^31-1]
#define INTEGER32_MAX (2147483647)
#define INTEGER32_MIN (-2147483648)

int Atoi(const char *str) {
  unsigned int value = 0;
  // 1表示负数，0表示整数
  int negative = 0;

  //判断str指针非NULL
  if (str != NULL) {
    //判断是否为负数
    if (str[0] == '-') {
      negative = 1;
      str++;
    }
    else if (str[0] == '+') {
      str++;
    }
    // str指向的字符串以0结尾
    while (str[0] != '\0') {
      if (str[0] >= '0' && str[0] <= '9') {
        //判断value是否会溢出
        if (value > UINTEGER32_MAX / 10) return 0;
        value *= 10;
        //判断value是否会溢出
        if (value > UINTEGER32_MAX - (str[0] - '0')) return 0;
        value += (str[0] - '0');
        str++;
      } else {
        //含有非法字符，value的值返回0
        value = 0;
        break;
      }
    }
    if (str[0] == '\0') {
      if (negative)  //负数
      {
        // INT32_MINI = -(INT32_MAX+1)
        if (value > (unsigned int)(INTEGER32_MAX + 1)) return 0;
        return -value;
      } else {  //正数
        if (value > INTEGER32_MAX) return 0;
        return value;
      }
    }
  }
  return value;
}

```

<!--more-->

- itoa函数实现

```bash

#define INT_MIN (-2147483648)
#define INT_MAX ( 2147483647)

unsigned digits10(unsigned v) {
    if (v < 10) return 1;
    if (v < 100) return 2;
    if (v < 1000) return 3;
    if (v < 1000000000) {
        if (v < 100000000) {
            if (v < 1000000) {
                if (v < 10000) return 4;
                return 5 + (v >= 100000);
            }
            return 7 + (v >= 10000000);
        }
        return 9;
    }
    return 10;
}


int Itoa(char *dst, unsigned dstlen, int svalue) {
    static const char digits[201] =
        "0001020304050607080910111213141516171819"
        "2021222324252627282930313233343536373839"
        "4041424344454647484950515253545556575859"
        "6061626364656667686970717273747576777879"
        "8081828384858687888990919293949596979899";
    int negative;
    unsigned value;

    if (svalue < 0) {
        value = -svalue;
        negative = 1;
    } else {
        value = svalue;
        negative = 0;
    }

    /* Check length. */
    unsigned const length = digits10(value)+negative;
    if (length >= dstlen) return 0;

    /* Null term. */
    unsigned next = length;
    dst[next] = '\0';
    next--;
    while (value >= 10) {
        int const i = (value % 100) * 2;
        value /= 100;
        dst[next] = digits[i + 1];
        dst[next - 1] = digits[i];
        next -= 2;
    }

    /* Handle last 1 digits. */
    if (value > 0 || length == 1 ) {
        dst[next] = '0' + value;
    }

    /* Add sign. */
    if (negative) dst[0] = '-';
    return length;
}

```
