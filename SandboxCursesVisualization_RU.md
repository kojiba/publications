
<img src="https://habrastorage.org/getpro/habr/post_images/503/5b9/b96/5035b9b96e4f7b2f5573a2c42fbd7c25.png" alt="image"/>

Эта статья является логичным продолжением моей старой статьи про визуализацию работы brainfuck алгоритмов на виртуальной машине с циклической памятью. Под катом немного тяжеловатого видео и картинок.

Почему debug через print? Print является самым простым средством визуализации процессов и памяти, хорош тем, что он работает вместе с написанным кодом (в отличие от отладчика, который например как в swift просто не воспринимает иногда breakpoints, evals и проч привычные вещи)
А теперь представим, что всю работу программы можно визуализировать через print? Казалось бы нет, но если программа не занимает довольно много памяти, то можно.

Конечно, такая визуализация не будет полной, без визуализации регистров процессора, всего стека, кучи, и маленького slow-down работы этой программы. 
В <a href="https://github.com/kojiba/RayLanguage">RayFoundation</a> на уровне библиотеки заложена возможность подменять memory managment указатели (как malloc, free), и таким образом контролировать и/или sandbox-ить код, об этом немного <a href="https://habrahabr.ru/post/244617/">тут</a>, но если не читали - не отвлекайтесь, потому что такой большой обьем технических подробностей тут не очень уместен (интересен будет не всем).

Так вот, можно считать что sandbox создает непрерывную страницу памяти какого-то, заданного программистом размера. Как оказалось, для тестов ray было достаточно (для красоты визуализации это должен быть квадрат NxN) 256 х 256 = 65536 байт памяти + размер worker структуры самой sandbox. И это для работы в "простом" и "<a href="https://en.wikipedia.org/wiki/Address_space_layout_randomization">ASLR</a>" режимах. Далее будет наглядно показана разница в этих режимах, через тот самый master-lvl-print-дебаг.

Несложными манипуляциями, прикручивая ncurses библеотеку, можно получить один из интересных способов визуализации, для настоящих линуксоидов/POSIXоидов - визуализации в терминале через текстовую графику. Исходные коды больших визуализаций лежат в <a href="https://github.com/kojiba/RayLanguage/commit/23ced975c0dec86bd3a3ba1a6a7a2c86a8534b87">этом коммите</a>.

<h4>Что нужно сделать?</h4>
<ul>
	<li>Сделать sandbox и выбрать ему размер </li>
	<li>Сделать делеи в работе программы (в данном случае это делеи во время malloc, calloc, realloc, free потому что это единственные контрольные точки, изменение которых не расползется по всему коду, можно было вставить usleep везде, но это для тех кому не будет лениво, но если это inject-библиотека для анализа, то это именно единственные контрольные точки для аналитика, и поставить usleep везде нет возможности)</li>
        <li>Отображать память sandbox в терминале через ncurses в отличном от main потоке </li>
</ul>

Сказано - сделано. Первые результаты (смотреть в полноэкранном режиме):

<h4>Default-mode</h4>  

[![DefaultModeVideo](http://img.youtube.com/vi/oTtGlyUQRZQ/0.jpg)](http://www.youtube.com/watch?v=oTtGlyUQRZQ)

(каждая цветная точка - это байт памяти со значением >0, нулевые байты черного цвета, немного криво от того что терминал размером 512х256 мне так и не удалось создать с нормальным масштабом)

Это работа в дефолтном моде, где каждый элемент который был malloc-цирован ставится в самое начало свободного участка памяти.
Я, как человек который знает что делает код, могу на "глаз" сказать где какая структура находится, потому что для ray, например, характерно наличие autoPool, который чистит память если нужно, это массив, который хранит все указатели на malloc-ированные обьекты. Получается такой себе мешок в мешке, т.к. этот пул работает внутри sandbox, и я долго добивался эффекта, чтобы можно было сделать множество вложенных пулов и sandbox, но это лирика. Приблизительная (немного грубо отмечено) схема обьектов:

<img src="https://habrastorage.org/getpro/habr/post_images/04c/e3c/e86/04ce3ce867d1f398107d189714ce93ef.jpg" alt="image"/>

Круто же, неправда ли? 
Можно посмотреть как выглядит память, попробовать себя на статическом и динамическом анализе в отличении отдельных обьектов, структур хранения. Но случай выше очень идеальный, для реального мира. Дело в том, что sandbox при каждом удалении обьектов делал memset(0), что почти никогда, реальный аллокатор не делает, он делает это после определенного количества освобожденных обьектов и делает это целыми страницами.

Вот пример стандартной sandbox с "насыщением" т.е. место удаленных обьектов переиспользуется а не зануляется:

<h4>Default-mode with saturation</h4>

[![ASLRModeVideo](http://img.youtube.com/vi/WoVQD1EumaM/0.jpg)](http://www.youtube.com/watch?v=WoVQD1EumaM)

Для систем требовательных к безопасности здесь можно отметить несколько особенностей:
<ul>
	<li>некоторые обьекты которые находятся в памяти достаточно долго все еще можно отследить (RPool в верху экрана) </li>
	<li>некоторая критическая информация может остаться не удаленной и ее можно считать</li>
        <li>равномерное замусоривание в области работы не дает четко распознать какие обьекты там работают  </li>
        <li>можно провести частичный анализ по области значений байт, и попытаться выделить основные рабочие структуры, но замусоривание делает это чуть сложнее</li>
</ul>

<h4>ASLR-mode</h4>  

[![ASLRModeVideo](http://img.youtube.com/vi/_yDivYCJ8WM/0.jpg)](http://www.youtube.com/watch?v=_yDivYCJ8WM)

Еще круче. Но, в этой картинке трудно что-то разобрать. Единственное что я увидел это одну активную область, которая постоянно изменяется. Что это может быть за область? Она всегда меняется и остается на одном и том же месте.
<img src="https://habrastorage.org/getpro/habr/post_images/ee0/7dc/90c/ee07dc90c7483b37c20161fa69cb58b7.png" alt="image"/>

<details closed>
    <summary>ответ</summary>
    <pre>это счетчик RPool-а, который знает сколько указателей находится в текущий момент времени в программе на куче, и рядом с ним счетчик свободных мест, которые остаются для указателей</pre>
</details>

Можно выделить еще немного особенностей для ASLR-mode
<ul>
	<li>при достаточной архитектуре обьектов с указателями и интенсивной работе с кучей довольно тяжело разобрать что здесь где находится, т.к. одна статическая часть обьекта может быть в одном куске памяти, а остальные суб-обьекты могут быть совершенно в другом</li>
	<li>по времени жизни обьектов и интенсивности изменения их кусков все еще можно распознать некоторые части программы</li>
        <li>равномерное распределение памяти выглядит безопаснее от того, что не зная значения ГПСП нельзя провести атаку по конкретному указателю обьекта, а если обьект использует realloc, и не зная времени этого realloc - нельзя провести атаку длительную по времени</li>
        <li>можно частичный анализ по range значений байт, и попытаться выделить основные рабочие структуры, но при множественных суб-частях и realloc-ах почти нереально</li>
</ul>

<h4>ASLR-mode with saturation</h4>

[![ASLRModeVideoSaturation](http://img.youtube.com/vi/TNy9PvZ3OQk/0.jpg)](http://www.youtube.com/watch?v=TNy9PvZ3OQk)

В конце видео космический мусор. Особенности этого мода похожи на особенности default мода с насыщением, но из-за большой интенсивности работы в куче вообще трудно что-либо разобрать, кроме той самой активной области с предыдущего скрина, но если бы в какой то момент эта структура realloc-нулась то считай и это было бы потеряно.

Есть еще пару видео с визуализацие работы тестов RList, на меньшем куске памяти и с большим масштабом.

default-mode:

[![ASLRModeVideo](http://img.youtube.com/vi/im-wuL6-XKM/0.jpg)](http://www.youtube.com/watch?v=im-wuL6-XKM)

aslr-mode:

[![ASLRModeVideo](http://img.youtube.com/vi/OWPvN9efLmk/0.jpg)](http://www.youtube.com/watch?v=OWPvN9efLmk)

<h4>Выводы</h4>
Такой инструмент наглядно показывает работу структур, позволяет находить утечки памяти, а alsr иногда помогает находить access violation, т.к. если переписать чуть дальше своего размера можно внезапно получить sigsegv и много интересных вещей. С другой стороны это еще и помогает своему коду быть немного безопаснее.
