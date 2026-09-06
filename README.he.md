<div dir="rtl">

# E22 LoRa UART

[English](README.md)

ספריית ארדואינו למודולי ה-LoRa של Ebyte מסדרת E22 עם ממשק UART. אנחנו משתמשים בה בכרטיס
התקשורת של הלוויין, שיש בו ATmega328PB ומודול E22-400T30D. היא עובדת גם על ESP32, שנוח יותר
לעבודה על השולחן.

זה רק הדרייבר של המודול. הלוגיקה של הביקון והמשיב שרצה על הלוויין היא sketch נפרד שמשתמש
בספרייה הזאת.

## על המודול

ה-E22 הוא לא שבב LoRa עם ממשק SPI. זה לוח קטן עם מיקרו-בקר משלו שיושב לפני שבב רדיו SX1268.
מדברים עם המיקרו-בקר הזה דרך פורט טורי ב-9600 באוד. חוץ מזה יש שלושה פינים בשימוש: M0 ו-M1
בוחרים את מצב העבודה, ו-AUX מראה אם המודול עסוק (LOW) או פנוי (HIGH). אין NSS, אין DIO0,
ואין גישה ישירה לרגיסטרים של הרדיו.

למודול ה-DIP יש שבעה פינים:

| פין | שם | הערות |
|-----|------|------------------------------------------------------------|
| 1 | M0 | ביט 0 של המצב. אסור להשאיר צף. |
| 2 | M1 | ביט 1 של המצב. אסור להשאיר צף. |
| 3 | RXD | כניסה טורית, לוגיקה של 3.3V |
| 4 | TXD | יציאה טורית |
| 5 | AUX | יציאת עסוק/פנוי. לא לנהוג אותו. |
| 6 | VCC | 5V להספק מלא. בשידור ה-T30D מושך עד 620mA. |
| 7 | GND | |

מצבי עבודה, נבחרים עם M1 ו-M0:

- `0 0` שידור. שליחה וקבלה רגילות.
- `0 1` wake-on-radio. לא בשימוש אצלנו.
- `1 0` תצורה. במצב הזה הפורט הטורי הוא תמיד 9600 8N1.
- `1 1` שינה.

התצורה היא תשעה בייטים של רגיסטרים: כתובת, מזהה רשת, קצב באוד וקצב אוויר, גודל פקטה והספק,
ערוץ, ביטים של אפשרויות, ומפתח הצפנה. קוראים אותם עם `C1 addr len`. כותבים אותם עם
`C0 addr len data` (נשמר בפלאש) או `C2 addr len data` (זמני). בסעיף 6 של המדריך שב-`docs/` יש את
מפת הרגיסטרים המלאה.

כמה עובדות על המודול שמשפיעות על התכנון:

- אי אפשר לקבוע את ה-Spreading Factor ישירות. למודול יש "קצב אוויר" בין 2.4 ל-62.5 kbps, וכל
  קצב הוא צירוף SF/BW קבוע ש-Ebyte בחרו. `setAirRate()` היא השליטה היחידה שיש לנו.
- הצד השני של הקישור כנראה צריך להיות גם E22. Ebyte מוסיפים מעל LoRa פרוטוקול ו-FEC משלהם,
  ולא מצאנו מישהו שפענח את זה עם SX127x רגיל. דוד אומר שקלט את זה פעם עם חומרה אחרת. נבדוק
  את זה במקום להניח.
- התדר הוא מספר ערוץ: 410.125 MHz ועוד ערוץ × 1 MHz, ערוצים 0 עד 83. המשדר-מקלט הראשי של
  הלוויין נמצא על 436.4 MHz, אז אסור להשתמש בערוצים 25 עד 28. הערוץ הוא הגדרה בזמן ריצה, לא
  קבוע.
- המודול לא שומר על גבולות של פקטות בצד הטורי. שתי פקטות שנשלחו אחת אחרי השנייה יכולות לצאת
  מה-UART כזרם אחד. פורמט הפקטה שלנו צריך בייטים של סנכרון, שדה אורך ו-checksum משלו.

## שימוש

<div dir="ltr">

```cpp
#include <E22.h>

E22 radio(Serial1, 4, 5, 6);   // serial port, M0, M1, AUX. Optional 5th argument: RESET pin.

void setup() {
  Serial.begin(115200);
  radio.begin(9600);

  E22Config cfg;
  radio.readConfig(cfg);
  cfg.channel = 23;            // 433.125 MHz
  cfg.rssiByte = true;         // the module adds an RSSI byte to each received packet
  radio.writeConfig(cfg);      // writes, reads back, compares

  radio.send("hello\n");
}

void loop() {
  int b = radio.readByte();
  if (b >= 0) Serial.write(b);
}
```

</div>

כל קריאת תצורה מעבירה את המודול למצב תצורה, מבצעת את הפעולה, ומחזירה אותו. לכל המתנה על AUX
יש timeout, והקריאה מחזירה `false` אם הוא פג. שום דבר בספרייה לא נתקע לנצח. זה חשוב כי הדבר
היחיד שה-OBC יכול לעשות לכרטיס הזה הוא לכבות ולהדליק אותו.

הספרייה רק מעבירה בייטים. מסגור (framing), checksum ומונים שייכים ל-sketch שמשתמש בה.

### דוגמאות

להריץ בסדר הזה על לוח חדש:

1. `ReadConfig` מדפיס את כל הרגיסטרים ואת גרסת הקושחה. אם זה עובד, החיווט נכון.
2. `PingPong` צריך שני לוחות. אחד שולח PING, השני עונה PONG. שניהם מדפיסים RSSI.
3. `AirRateSweep` הוא ניסוי "שינוי SF כל 10 שניות" מהמסמך של הפרויקט, עם קצב אוויר במקום.
   שני הלוחות עוברים על ששת הקצבים בסנכרון, מרגע ההדלקה.

בכל דוגמה יש בהתחלה בלוק שבוחר את הפורט הטורי ואת הפינים לכל יעד. מספרי הפינים של ה-ATmega
שם הם ערכי מקום עד שתהיה לנו הסכמה.

### API

| קריאה | תיאור |
|------|-------------|
| `begin(baud, rx, tx)` | פותח את הפורט, נכנס למצב שידור, מחכה ל-AUX. rx/tx בשימוש רק ב-ESP32. |
| `setMode(E22Mode)` | קובע M0/M1, מחכה ל-AUX, ומעביר את הפורט הטורי ל-9600 במצב תצורה |
| `waitIdle(timeoutMs)` | מחכה עד ש-AUX גבוה |
| `hardReset()` | נותן פולס ל-RESET, אם ניתן פין RESET. למודול ה-DIP אין פין RESET. |
| `readConfig(cfg)` / `writeConfig(cfg, persist)` | כל תשעת הרגיסטרים. `persist=false` משתמש ב-`C2`, אז השינוי נמחק אחרי כיבוי. |
| `setChannel`, `setAirRate`, `setTxPower`, `setAddress`, `setRssiByte`, `setAmbientRssi` | שינוי של שדה אחד |
| `atCommand(cmd, reply, len)`, `readFirmwareVersion(buf, len)` | פקודות AT, במצב תצורה |
| `send(buf, len)`, `send("text")` | שולח בחלקים בגודל פקטה, מחכה ל-AUX בין החלקים |
| `sendTo(addr, ch, buf, len)` | מוסיף את הכותרת של מצב fixed-point. דורש `cfg.fixedPoint`. |
| `available()`, `read()`, `readByte()`, `flushInput()` | בייטים גולמיים מהמודול |
| `readAmbientRssi(dbm)`, `readLastPacketRssi(dbm)` | דורשים `cfg.ambientRssi`, רק במצב שידור |
| `E22::rssiByteToDbm(b)` | ממיר את בייט ה-RSSI ל-dBm |

רמות הספק: `E22TxPower::Level0` היא המקסימום בכל מודול. `dBm30`..`dBm21` ו-`dBm22`..`dBm10` הם
שמות לאותם ארבעה קודים במודולים של 30 dBm ו-22 dBm.

## בנייה

ללוח הטיסה צריך את MiniCore, שמוסיף את ה-ATmega328PB:

<div dir="ltr">

```
arduino-cli config add board_manager.additional_urls https://mcudude.github.io/MiniCore/package_MCUdude_MiniCore_index.json
arduino-cli core install MiniCore:avr
arduino-cli compile --fqbn MiniCore:avr:328:variant=modelPB,clock=16MHz_external --library . examples/ReadConfig
```

</div>

ל-ESP32 על השולחן:

<div dir="ltr">

```
arduino-cli config add board_manager.additional_urls https://espressif.github.io/arduino-esp32/package_esp32_index.json
arduino-cli core install esp32:esp32
arduino-cli compile --fqbn esp32:esp32:esp32 --library . examples/ReadConfig
```

</div>

ה-CI מקמפל את הדוגמאות לשני היעדים בכל push.

## הערות ומגבלות

- ל-ATmega יש 2KB של RAM. להשתמש בבאפרים קטנים, לשים מחרוזות ב-`F()`, לא להשתמש ב-`printf`.
- ה-ATmega עובד ב-5V והכניסות של המודול הן 3.3V. הלוח חייב לעשות level shifting ל-RXD, M0 ו-M1.
  נראה שבסכמה יש נגדים טוריים על הקווים האלה. עדיין לא אושר.
- מצב תצורה הוא תמיד 9600 8N1. אם ה-UART רץ במהירות אחרת, הדרייבר פותח מחדש את הפורט לפני
  ואחרי כל קריאת תצורה.
- אם AUX מוחזק LOW בזמן שהמודול נדלק, המודול נכנס למצב עדכון קושחה ומפסיק להגיב. אף פעם לא
  לשים pull-down על AUX.
- ב-30 dBm המודול מושך עד 620mA בזמן שידור. אם ה-5V נופל, המודול מתאפס באמצע פקטה.
- בקושחה 7453-0-21 ומעלה, AUX לא יורד ל-LOW בזמן השידור ברדיו אלא אם `AT+UAUX` מופעל. אז
  `send()` חוזרת כשהבאפר הטורי ריק, לא כשהפקטה שודרה.

## מצב

מתקמפל לשני היעדים. עדיין לא נבדק על חומרה אמיתית. נכתב לפי המדריך, ופריסת הבייטים הושוותה
מול [הספרייה ל-E22](https://github.com/xreef/EByte_LoRa_E22_Series_Library) של Renzo Mischianti.
הבדיקה הראשונה על לוח אמיתי: להריץ `ReadConfig` ולבדוק שערכי המפעל `00 00 00 62 00 17 03 00 00`
חוזרים, ואז לרשום את גרסת הקושחה.

רישיון MIT.

</div>
