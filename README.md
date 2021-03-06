# Скрипт импорта товаров из YML в OpenCart 3

## Окружение
- Язык: **PHP 7.2**
- Источник товаров: **YML поставщика**
- БД: **MySQL 5.6.39-83.1**
- Веб-сервер: **Apache/2.2.15 (CentOS)**
- Версия клиента базы данных: **libmysql - 5.5.35**
- PHP расширение: **mysqli, Imagick**
- Кодировка сервера: **UTF-8 Unicode (utf8)**

## Краткое описание
Скрипт группирует предложения поставщика по одинаковым или похожим картинкам и формирует товары, опции и значения опций по принципам:
- 1 группа предложений <=> 1 товар
- 1 предложение <=> 1 значение опции товара
Взаимосвязь между предложениями и товарами обусловлена хранением артикулов поставщиков в поле `meta_keyword` товара.
Модератор сайта может вручную перегруппировывать опции товаров, перемещая артикулы предложений поставщиков из одного товара в другой.
Сопостовление категорий реализовано по наименованием и синонимам категорий. Синонимы хранятся в поле `meta_keyword` категории.
Не распознанные категории поставщика и товары, относящиеся к этим категориям грузятся в выключенную категорию **Неотсортированные при импорте**.

## Описание алгоритма
1. Получение данных от поставщика в формате YML, конвертация в ассоциативный массив, где ключ - md5 hash первой картинки предложения, либо очень похожей на первую картинку, таким образом происходит первичная группировка по одинаковым и схожим картинкам.
2. Для получения возможности сопоставления русского текста после его упрощения, во избежании случаев с задвоением пробелов, например, данные по товарам, атрибутам, категориям, фильтрам, производителям, опциям и значениям опций забираются из БД и превращаются в "поисковые индексы", где индексами является результат функции `str2idx` (см. ниже), на каждую сущность создаются индексы по ID, в которых хранятся сами данные и отдельные индексы по наименованию и синонимам (если есть), которые хранят индексы ID.
3. Сопоставление полученных от поставщика групп предложений и существующих товаров и их опций. Связующее поле здесь `product["meta_keyword"]`, в которое через точку с запятой вписываются артикулы предложений, состовляющих опции товара, либо сам товар, если соответствующее предложение для него одно. Это позволяют пользователю вручную перегруппировать предложения по товарам, соответственно, на данном этапе, необходимо учесть изменения от пользователей.
4. Конвертация группы предложений в товар. Здесь проверяется:

 	4.1 Наличие действующей категории для товара, если таковой не найдено, то товар, вместе с категорией поставщика отправляется в выключенную категорию "Неотсортированные при импорте". Позднее менеджер может указать в поле `category["meta_keyword"]` у действующей категории, через точку с запятой наименования категорий поставщика, что будет расцениваться как синонимы при следующем поиске действующей категории, что позволит автоматически сопоставить категории.

 	4.2 Наличие фильтров и групп фильтров, подходящих к параметрам, имеющимся у каждого из предложений. Отстутствие подходящих к параметрам групп фильтров, последние не создаются, также не создаются и сами фильтры. Если же подхожящая группа фильтров найдена, то при отсутствии подходящих фильтров - они создаются.

	4.3 Наличие атрибутов и групп атрибутов, опять же подходящих к параметрам. В отличие от фильтров создаются и группы и сами атрибуты, если не были найдены подходящие.

	4.4 Наличие подходящей по наименованию опции товара. Наименование опции товара составляется из наименований различающихся параметров, у которых отброшено все, что поле `,`. Если различных параметров несколько, например, **Размер** и **Вес наполнителя**, то они записываются в одну строку через раделитель ` / `. В итоге получается **Размер / Вес наполнителя**. В случае отсутствия подходящей опции - она создается.

	4.5 Наличие подходящих к значениям параметров значений опций. Формируеются аналогично наименованиям опций, но из значений различющихся параметров. В случае отсутствия - создаются.

	4.6 Формирование общей цены, по принципу наименьшей. У каждой опции в свою очередь сохраняется наценка, расчитываемая по формуле *цена предложения - общая цена*.
	
    4.7 Формирование общего остатка, путем агрегации остатков по все предложениям. У каждой опции, в свою очередь, остаются сведения об остатках конретно по ней.
	
    4.8 Формирование перечня фотографий. Перебираются фотографии всех предложений, выбираются уникальные.

## Сокращения и определения
- *prod* - product
- *opt* - option
- *val* - value
- *cat* - category
- *man* - manufacturer
- *idx* - index
- *desc* - description
- *cur* - current 
- *Индекс* - строка и или число, привиденного к типу `string`, из которого убраны пробелы и добавлен префикс "_"