# ДЗ 13
## Создать триггер для поддержки витрины в актуальном состоянии.

Для начала создал все необходимые таблицы в соответствии с приложенным файлом и заполнил соответсвующими данными.

после создания таблицы-витрины - перед созданием триггеров заполнил её уже имеющимися записями:
```
INSERT INTO good_sum_mart SELECT G.good_name, sum(G.good_price * S.sales_qty) AS sum_sale
  FROM goods G
  INNER JOIN sales S ON S.good_id = G.goods_id
  GROUP BY G.good_name;
```

Добавил следующие триггерные функции:

При добавлении следует добавление текущей стоимости записи, либо добавление новой записи в таблицу со значением записи
```
CREATE or replace function ft_insert_sales()
RETURNS trigger
AS
$TRIG_FUNC$
DECLARE
g_name varchar(63);
g_price numeric(12,2);
BEGIN
SELECT G.good_name, G.good_price*NEW.sales_qty INTO g_name, g_price FROM goods G where G.goods_id = NEW.good_id;
IF EXISTS(select from good_sum_mart T where T.good_name = g_name)
THEN UPDATE good_sum_mart T SET sum_sale = sum_sale + g_price where T.good_name = g_name;
ELSE INSERT INTO good_sum_mart (good_name, sum_sale) values(g_name, g_price);
END IF;
RETURN NEW;
END;
$TRIG_FUNC$
LANGUAGE plpgsql
VOLATILE
SET search_path = pract_functions, public
COST 50;
```
При удалении следует вычитание текущей стоимости записи с последующим удалением строк, у которых стоимость меньше либо равна 0
```
CREATE or replace function ft_delete_sales()
RETURNS trigger
AS
$TRIG_FUNC$
DECLARE
g_name varchar(63);
g_price numeric(12,2);
BEGIN
SELECT G.good_name, G.good_price*OLD.sales_qty INTO g_name, g_price FROM goods G where G.goods_id = OLD.good_id;
IF EXISTS(select from good_sum_mart T where T.good_name = g_name)
THEN 
UPDATE good_sum_mart T SET sum_sale = sum_sale - g_price where T.good_name = g_name;
DELETE FROM good_sum_mart T where T.good_name = g_name and (sum_sale < 0 or sum_sale = 0);
END IF;
RETURN NEW;
END;
$TRIG_FUNC$
LANGUAGE plpgsql
VOLATILE
SET search_path = pract_functions, public
COST 50;
```
При обновлении следуют обе процедуры описанные выше.
```
CREATE or replace function ft_update_sales()
RETURNS trigger
AS
$TRIG_FUNC$
DECLARE
g_name_old varchar(63);
g_price_old numeric(12,2);
g_name_new varchar(63);
g_price_new numeric(12,2);
BEGIN
SELECT G.good_name, G.good_price*OLD.sales_qty INTO g_name_old, g_price_old FROM goods G where G.goods_id = OLD.good_id;
SELECT G.good_name, G.good_price*NEW.sales_qty INTO g_name_new, g_price_new FROM goods G where G.goods_id = NEW.good_id;
IF EXISTS(select from good_sum_mart T where T.good_name = g_name_new)
THEN UPDATE good_sum_mart T SET sum_sale = sum_sale + g_price_new where T.good_name = g_name_new;
ELSE INSERT INTO good_sum_mart (good_name, sum_sale) values(g_name_new, g_price_new);
END IF;
IF EXISTS(select from good_sum_mart T where T.good_name = g_name_old)
THEN 
UPDATE good_sum_mart T SET sum_sale = sum_sale - g_price_old where T.good_name = g_name_old;
DELETE FROM good_sum_mart T where T.good_name = g_name_old and (sum_sale < 0 or sum_sale = 0);
END IF;
RETURN NEW;
END;
$TRIG_FUNC$
LANGUAGE plpgsql
VOLATILE
SET search_path = pract_functions, public
COST 50;
```
Добавил следующие триггеры:
```
CREATE TRIGGER tr_insert_sales
AFTER INSERT
ON sales
FOR EACH ROW
EXECUTE PROCEDURE ft_insert_sales();

CREATE TRIGGER tr_delete_sales
AFTER DELETE
ON sales
FOR EACH ROW
EXECUTE PROCEDURE ft_delete_sales();

CREATE TRIGGER tr_update_sales
AFTER UPDATE
ON sales
FOR EACH ROW
EXECUTE PROCEDURE ft_update_sales();
```

Можно было так же сделать это при помощи одной триггерной функции с ветвлением по TG_OP, но я решил что читаемее будет разделить это на 3 отдельные функции.

## Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?

Благодаря тому, что изменение цен идет инкрементивным способом, а не вычисляется каждый раз с нуля - мы сохраняем общие стоимости всех транзакций на момент их совершения.
Но при текущей схеме БД реализовать корректное удаление из стоимости от витрины - не представляется возможным, т.к. для вычитания мы используем текущее значение стоимости, а не стоимость на момент транзакции.

Возможно потребуется расширить таблицу sales колонкой стоимости единицы на момент заключения договора купли-продажи, что бы сумма сделки корректно высчитывалась и вычиталась из витрины.
