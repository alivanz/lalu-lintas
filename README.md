# lalu-lintas
Sistem Pengaturan Lalu Lintas dengan Mikro-Prosesor

# Solusi dengan Rangkaian Mikro-Kontroler

## Mikro-Kontroler
Pada solusi yang ditawarkan kali ini, akan digunakan Chip ATMega328P yang terinstall Bootloader Arduino.
![Pin ATMega328P](http://nearbus.net/wiki/images/5/5f/ATM328_arduino_pinout.jpg)

## Konfigurasi LED RGB
Untuk menyatakan ketiga warna rambu lalu lintas, digunakan lampu LED RGB (Red-Green-Blue). Warna merah dinyatakan dengan lampu Red, warna hijau dinyatakan dengan lampu Green, sedangkan warna kuning dinyatakan dengan kombinasi lampu Red dan Green.
Pada kali ini, digunakan lampu LED RGB Common Cathode. Tiap-tiap pin lampu (kecuali pin common cathode) dihubungkan pada output digital dari mikro-kontroler. Untuk membatasi arus yang mengalir pada LED, diberikan resistor sebesar 1kOhm sebelum terhubung dengan mikro-kontroler.
![Pin LED RGB Common Cathode](http://www.nkcelectronics.com/assets/images/rgb_5mm_cathode.jpg)

Pada kali ini, pin-pin yang digunakan untuk men-drive nyala lampu LED adalah sebagai berikut,
|       |RED  |GREEN  |BLUE |
|-------|-----|-------|-----|
|NORTH	|PC0	|PC1	  |PC2  |
|EAST	  |PC3	|PC4	  |PC5  |
|SOUTH	|PD4	|PD5    |PD6  |
|WEST   |PB0	|PB1    |PB2  |
Pin-pin yang dipilih berdasarkan beberapa hal yaitu dengan menghindari pin yang digunakan untuk komunikasi serial (TX, RX,) untuk “future use” dan pin untuk Interrupt pejalan kaki (INT0).
Untuk memudahkan dalam mengatur warna, dibuat konstanta-konstata warna nyala lampu LED bersesuaian dengan urutan pin warna lampu LED. Pada kasus ini terurut warna Red-Green-Blue. Untuk warna selain merah, hijau, dan biru dilakukan kombinasi penyalaan lampu untuk mendapatkan persepsi warna yang lebih variatif. Konstanta tersebut seperti pada berikut,

```C
/*                COLOR                 */  
const uint16_t RED    = 1;  
const uint16_t GREEN  = 2;  
const uint16_t BLUE   = 4;  
const uint16_t YELLOW = RED|GREEN;  
const uint16_t PURPLE = RED|BLUE;  
const uint16_t CYAN   = GREEN|BLUE;  
const uint16_t WHITE  = RED|GREEN|BLUE;  
```

## State
Dari 4 arah kendaraan, hanya boleh ada satu jalur yang berjalan setiap waktu, sedangkan sisanya menunggu gilirannya masing-masing. Artinya dalam satu waktu, hanya ada satu lampu berwarna hijau atau kuning, tiga sisanya haruslah berwarna merah. Selain itu ada tombol interupsi untuk mempercepat durasi waktu hijau (durasi kuning tidak terpengaruh) guna memfasilitasi pejalan kaki untuk lewat dengan lebih cepat (in case, apabila pejalan kaki terburu-buru?). Hasilnya, adalah 12 state yang terdefinisi sebagai berikut,

STATE |	DURATION | NEXT STATE	NEXT STATE ON INTERRUPT	| NORTH |	EAST | SOUTH | WEST
--- | --- | --- | --- | --- | --- | ---
0	|10s|1	|8	|Green	|Red	|Red	|Red
1	|1s	|2	|1	|Yellow	|Red	|Red	|Red
2	|10s|3	|9	|Red	  |Green	|Red	|Red
3	|1s	|4	|3	|Red	  |Yellow	|Red	|Red
4	|10s|5	|10	|Red	  |Red	|Green	|Red
5	|1s	|6	|5	|Red	  |Red	|Yellow	|Red
6	|10s|7	|11	|Red	|Red	|Red	|Green
7	|1s	|0	|7	|Red	|Red	|Red	|Yellow
8	|5s	|1	|8	|Green	|Red	|Red	|Red
9	|5s	|3	|9	|Red	|Green	|Red	|Red
10|5s	|5	|10	|Red	|Red	|Green	|Red
11|5s	|7	|11	|Red	|Red	|Red	|Green

Dari tabel state diatas, dapat diimplementasikan kedalam kode C seperti dibawah ini,

```C
struct TState{  
  uint16_t lamp[4];  
  uint16_t interval;  
  int next;  
  int onInterrupt;  
};  
const struct TState states[] = {  
  {{GREEN,RED,RED,RED},   10, 1,  8},  
  {{YELLOW,RED,RED,RED},  1,  2,  1},  
  {{RED,GREEN,RED,RED},   10, 3,  9},  
  {{RED,YELLOW,RED,RED},  1,  4,  3},  
  {{RED,RED,GREEN,RED},   10, 5,  10},  
  {{RED,RED,YELLOW,RED},  1,  6,  5},  
  {{RED,RED,RED,GREEN},   10, 7,  11},  
  {{RED,RED,RED,YELLOW},  1,  0,  7},  
  {{GREEN,RED,RED,RED},   5,  1,  8},  
  {{RED,GREEN,RED,RED},   5,  3,  9},  
  {{RED,RED,GREEN,RED},   5,  5,  10},  
  {{RED,RED,RED,GREEN},   5,  7,  11},  
};
```

Untuk memudahkan dan mencegah kesalahan dalam peralihan state, menggunakan fungsi dapat menjadi salah satu solusi. Dalam fungsi tersebut selain melakukan perubahan state dan perhitungan durasi, juga melakukan pengaturan warna lampu.

```C
int state;  
#define CSTATE states[state]  
uint16_t count;  

void toState(int x){  
  state = x;  
	count = 1;  
  PORTC = CSTATE.lamp[1]<<3 | CSTATE.lamp[0];  
  PORTD&= 0x0F;  
  PORTD|= CSTATE.lamp[2]<<4;  
  PORTB = CSTATE.lamp[3];  
}  

ISR(TIMER1_COMPA_vect){  
  if(count<CSTATE.interval){  
    count++;  
  }else{  
    toState(CSTATE.next);  
  }  
  // Blink  
  PORTB |= 1<<5;  
  _delay_ms(100);  
  PORTB &= 0xFF ^ 1<<5;  
}  
ISR(INT0_vect){  
  state = CSTATE.onInterrupt;  
}  
```

## Main Program
Pada main program, terjadi pengaturan timer, pengaturan interrupt, dan pastinya pengaturan I/O pin. Setelah itu, program utama tidak melakukan apa-apa. Seluruh jalannya pengaturan lampu rambu lalu-lintas bekerja pada sub-routine interrupt (External Interrupt dan Timer/Counter Interrupt).

```C
int main(){  
  // PORT C&D as Output
  DDRB = 0xFF;
  DDRC = 0xFF;
  DDRD = 0xFF;  

  // Blink  
  PORTB = 0;  

  // TIMER1: Lamp Control  
  // CTC  
  TCCR1A  = 0;  
  TCCR1B  = (1<<WGM12);  
  // prescaler 1024  
  TCCR1B |= (1<<CS12)|(1<<CS10);  
  // Enable Interrupt  
  TIMSK1  = 1<<OCIE1A;  
  // 1 sec  
  OCR1A   = 15625;  

  // INT0: Pedestrian Crossing  
  // Active Low, Falling edge  
  EICRA   = (1<<ISC01);  
  // Enable interrupt  
  EIMSK   = (1<<INT0);  
  // Input PIN  
  DDRD   ^= 1<<2;  
  PORTD  |= 1<<2;  

  sei();  

  toState(0);  
  while(1);  

  return 1;  
}  
```

# Implementasi

## Membuat PCB
Desain PCB untuk mikro-kontroler dilakukan dengan membuat clone PCB Arduino Uno. Pada implementasinya, juga dilakukan instalasi Bootloader Arduino untuk memudahkan dalam memprogram mikro-kontroler ini (I don’t wanna “brick” the micro-processor). Kristal osilator juga menggunakan frekuensi yang sama dengan Arduino Uno, 16MHz.

## MERANGKAI KOMPONEN PADA PCB

## BURN BOOTLOADER ARDUINO
Untuk meng-upload Bootloader Arduino, kali ini digunakan Arduino lain sebagai ISP (In-System Program). Sebenarnya, akan lebih baik apabila menggunakan device yang lebih proper seperti USB-ASP, namun kali ini memanfaatkan resource yang telah tersedia (meminjam Arduino Uno punya teman).
Rangkaian untuk mem-burn Bootloader dapat dilihat seperti gambar dibawah. Intinya adalah menghubungkan pin SCK, MOSI, dan MISO, kontrol untuk Reset, dan pastinya catu daya 5V dan kristal osilator.
![Menggunakan Board Arduino sebagai ISP](https://www.arduino.cc/en/uploads/Tutorial/BreadboardAVR.png)

## Upload The Code
Library yang dibutuhkan telah tersedia pada Arduino IDE. Pertama, adalah mengupload program ISP pada Arduino yang telah ada. Dapat dilakukan dengan cara memilih pada menu File > Examples > Arduino ISP > Arduino ISP lalu upload kode tersebut. Kemudian hubungkan seperti rangkaian diatas, lalu upload Bootloader pada menu Tools > Programmer > Arduino as ISP dan Tools > Burn Bootloader.
Setelah bootloader ter-install, prosedur untuk mengupload ke Arduino Clone seperti mengupload kode Arduino biasa. Namun, karena Arduino Clone yang dibuat ini tidak memiliki USB Decoder untuk komunikasi Serial yang dibutuhkan untuk mengupload Kode ke Arduino Clone, digunakan FTDI USB Serial.
