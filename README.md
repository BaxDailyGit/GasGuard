# GasGuard
### 가스 누출 탐지 시스템 | 대학교 시스템프로그래밍및실습 팀프로젝트


본 프로젝트의 목적은 I/O용 센서 디바이스 제어가 필요한 가스 누출 탐지 시스템을 RPi(Raspberry Pi) 환경에서 Sensor – Actuator를 관리하여 개발하는 것이다.
4대의 RPi를 통해 구현하며 Linux의 functions을 활용하여 관련 기능들을 개발한다.



## 개발 조건
- RPi 기반의 시스템을 기반으로 하는 임베디드 프로그램 제안
- RPi에서 Linux의 5 core functions (process, memory, file system, i/o, network
management)를 활용하는 응용 프로그램 개발
- 다수의 Sensor 사용
- SPI, I2C, PWM, GPIO 중 적어도 2개는 wiringpi 사용없이 구현한다.

## 핵심 기능:

1. 가스 센서 데이터 수신 및 처리
2. 가스 농도에 따른 LED 제어 (빨간색, 노란색, 초록색)
3. 가스 농도에 따른 부저 제어 (경고음 발생)
4. LCD를 통한 가스 농도 정보 표시
5. 소켓 통신을 통한 데이터 수신

## 기술 스택:

- C언어
- Linux
    - GPIO (General Purpose Input/Output) 제어
        - LED 제어를 위한 GPIO 출력 핀 설정 및 값 쓰기
    - PWM (Pulse Width Modulation) 제어 
        - 부저 제어를 위한 PWM 사용
    - 파일 I/O
        - /sys/class/gpio 디렉터리를 통한 GPIO 제어
        - /sys/class/pwm 디렉터리를 통한 PWM 제어
- 소켓 프로그래밍
    - Berkeley 소켓 인터페이스 사용
    - TCP 프로토콜을 이용한 서버-클라이언트 통신
    - 서버로부터 가스 센서 데이터 수신
- I2C (Inter-Integrated Circuit) 통신
    - LCD 디스플레이 제어를 위한 I2C 프로토콜 사용
    - I2C 디바이스 파일을 통한 데이터 읽기/쓰기
- 임베디드 시스템 프로그래밍
    - Raspberry Pi 보드 사용
    - ARM 아키텍처 기반 Linux 운영체제

## 결과

https://github.com/BaxDailyGit/GasGuard/assets/99312529/1eeae39e-e93f-427a-b6e6-28bf2ead1c85

### 사용한 센서 및 엑추에이터: 
- 가스센서
- RGB-LED
- 스피커(부저)
- LCD


### 본인 파트

1. GPIO(General Purpose Input/Output) 제어
   - LED를 제어하기 위해 Raspberry Pi의 GPIO 핀을 사용
   - 사용한 GPIO 핀 번호: 17(빨간색 LED), 18(노란색 LED), 23(초록색 LED)
   - GPIO 핀을 출력 모드로 설정하고, 해당 핀의 값을 제어하여 LED를 켜고 끄기

2. 파일 I/O를 이용한 GPIO 제어
   - Linux의 sysfs 인터페이스(/sys/class/gpio)를 통해 GPIO 핀을 제어
   - /sys/class/gpio/export 파일에 GPIO 번호를 입력 -> 해당 핀이 export
   - /sys/class/gpio/gpio{pin}/direction 파일에 "out"을 입력 -> 출력 모드로 설정
   - /sys/class/gpio/gpio{pin}/value 파일에 0 또는 1을 써서 LED를 끄거나 켜기

3. 소켓 프로그래밍을 통한 가스 센서 데이터 수신
   - 서버로부터 TCP 소켓 통신을 통해 가스 센서 데이터를 수신
   - 소켓 생성, 서버 주소 설정, 연결 후 read() 시스템 호출로 데이터를 수신
   - 수신한 데이터를 정수형으로 변환하여 가스 농도 값 얻기

4. 가스 농도 범위에 따른 LED 제어 로직
   - 서버로부터 수신한 가스 센서 값(0~1023 범위)에 따라 LED를 다르게 제어
   - 가스 농도가 701~1023이면 빨간색 LED
   - 가스 농도가 401~700이면 노란색 LED
   - 가스 농도가 400 이하이면 초록색 LED




```shell
$ echo 17 > /sys/class/gpio/export
$ echo 18 > /sys/class/gpio/export
$ echo 23 > /sys/class/gpio/export
```
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUFFER_SIZE 5

void error_handling(char *message)
{
    fputs(message, stderr);
    fputc('\n', stderr);
    exit(1);
}

// Function to control the LEDs based on gas concentration
void control_leds(int gas_concentration)
{
    // GPIO pin numbers for the LEDs
    int led1_pin = 17;
    int led2_pin = 18;
    int led3_pin = 23;

    // Export GPIO pins
    FILE *export_file = fopen("/sys/class/gpio/export", "w");
    if (export_file == NULL)
        error_handling("Failed to open export file");

    fprintf(export_file, "%d\n", led1_pin);
    fprintf(export_file, "%d\n", led2_pin);
    fprintf(export_file, "%d\n", led3_pin);

    fclose(export_file);

    // Set the direction of GPIO pins to output
    char gpio_direction[256];
    sprintf(gpio_direction, "/sys/class/gpio/gpio%d/direction", led1_pin);
    FILE *gpio_direction_file = fopen(gpio_direction, "w");
    if (gpio_direction_file == NULL)
        error_handling("Failed to open GPIO direction file");

    fprintf(gpio_direction_file, "out");

    fclose(gpio_direction_file);

    sprintf(gpio_direction, "/sys/class/gpio/gpio%d/direction", led2_pin);
    gpio_direction_file = fopen(gpio_direction, "w");
    if (gpio_direction_file == NULL)
        error_handling("Failed to open GPIO direction file");

    fprintf(gpio_direction_file, "out");

    fclose(gpio_direction_file);

    sprintf(gpio_direction, "/sys/class/gpio/gpio%d/direction", led3_pin);
    gpio_direction_file = fopen(gpio_direction, "w");
    if (gpio_direction_file == NULL)
        error_handling("Failed to open GPIO direction file");

    fprintf(gpio_direction_file, "out");

    fclose(gpio_direction_file);

    // Turn off all LEDs
    char gpio_value[256];
    sprintf(gpio_value, "/sys/class/gpio/gpio%d/value", led1_pin);
    FILE *gpio_value_file = fopen(gpio_value, "w");
    if (gpio_value_file == NULL)
        error_handling("Failed to open GPIO value file");

    fprintf(gpio_value_file, "0");

    fclose(gpio_value_file);

    sprintf(gpio_value, "/sys/class/gpio/gpio%d/value", led2_pin);
    gpio_value_file = fopen(gpio_value, "w");
    if (gpio_value_file == NULL)
        error_handling("Failed to open GPIO value file");

    fprintf(gpio_value_file, "0");

    fclose(gpio_value_file);

    sprintf(gpio_value, "/sys/class/gpio/gpio%d/value", led3_pin);
    gpio_value_file = fopen(gpio_value, "w");
    if (gpio_value_file == NULL)
        error_handling("Failed to open GPIO value file");

    fprintf(gpio_value_file, "0");

    fclose(gpio_value_file);

    // Determine which LED to turn on based on gas concentration
    if (gas_concentration >=701 && gas_concentration <= 1023)
    {
        sprintf(gpio_value, "/sys/class/gpio/gpio%d/value", led1_pin);
        gpio_value_file = fopen(gpio_value, "w");
        if (gpio_value_file == NULL)
            error_handling("Failed to open GPIO value file");

        fprintf(gpio_value_file, "1");

        fclose(gpio_value_file);
    }
    else if ( gas_concentration >=401 && gas_concentration <= 700 )
    {
        sprintf(gpio_value, "/sys/class/gpio/gpio%d/value", led2_pin);
        gpio_value_file = fopen(gpio_value, "w");
        if (gpio_value_file == NULL)
            error_handling("Failed to open GPIO value file");

        fprintf(gpio_value_file, "1");

        fclose(gpio_value_file);
    }
    else if (gas_concentration <=400 )
    {
        sprintf(gpio_value, "/sys/class/gpio/gpio%d/value", led3_pin);
        gpio_value_file = fopen(gpio_value, "w");
        if (gpio_value_file == NULL)
            error_handling("Failed to open GPIO value file");

        fprintf(gpio_value_file, "1");

        fclose(gpio_value_file);
    }
}

int main(int argc, char *argv[])
{
    int sock;
    struct sockaddr_in serv_addr;
    char buffer[BUFFER_SIZE];
    int str_len;

    if (argc != 3)
    {
        printf("Usage: %s <IP> <port>\n", argv[0]);
        exit(1);
    }

    sock = socket(PF_INET, SOCK_STREAM, 0);
    if (sock == -1)
        error_handling("socket() error");

    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_addr.s_addr = inet_addr(argv[1]);
    serv_addr.sin_port = htons(atoi(argv[2]));

    if (connect(sock, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) == -1)
        error_handling("connect() error");

    while (1)
    {
        str_len = read(sock, buffer, sizeof(buffer) - 1);
        if (str_len == -1)
            error_handling("read() error");

        buffer[str_len] = '\0'; // Null-terminate the received data

        int gas_concentration = atoi(buffer);
        printf("Gas concentration: %d ppm\n", gas_concentration);

        control_leds(gas_concentration);
    }

    close(sock);

    return 0;
}

```

#### 가스센서(서버), LCD,부저(클라이언트) 코드 추가 예정
