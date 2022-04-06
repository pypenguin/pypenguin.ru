---
layout: post
title:  "1С-Битрикс ограничение максимального количество товаров в заказе."
date:   2021-11-22 17:03:00 +0300
categories: 1C-Bitrix
---


В этой заметке реализуем возможность настройки максимального количество товаров в заказе!

![My helpful screenshot](/images/bitrix/bx_basket_button.png)

Работать будет так: Если количество товаров в корзине больше максимального - кнопка "Оформить заказ" перестанет быть активной и под кнопкой будет выведен текст уведомления. Как только количество товаров становится меньше или равно указанному кнопка снова становится активной. Все работает на лету, без перезагрузки страницы. Изменять доступное количество к заказу товара станет возможно через параметры компонента. 

![My helpful screenshot](/images/bitrix/bx_basket_disabled_button.png)

#### Добавляем параметр в корзину
Для начала скопируем шаблон корзины в шаблон своего сайта. У меня это: /local/templates/ШАБЛОН_САЙТА/components/bitrix/sale.basket.basket

{% highlight ruby %}
$arTemplateParameters['MAX_COUNT_ITEMS'] = array(
    'PARENT' => 'BASE',
    'NAME' => "максимальное количество товаров в корзине",
    'TYPE' => 'STRING',
    'DEFAULT' => '0'
);
{% endhighlight %}

Открываем, в шаблоне, файл .parameters.php и в массив параметров arTemplateParameters добавляем код указанный выше:

{% highlight ruby %}
$arTemplateParameters['COLUMNS_LIST_MOBILE'] = array(
	'PARENT' => 'VISUAL',
	'NAME' => GetMessage('CP_SBB_TPL_COLUMNS_LIST_MOBILE'),
	'TYPE' => 'LIST',
	'COLS' => 25,
	'SIZE' => 7,
	'MULTIPLE' => 'Y',
);

$arTemplateParameters['MAX_COUNT_ITEMS'] = array(
    'PARENT' => 'BASE',
    'NAME' => "максимальное количество товаров в корзине",
    'TYPE' => 'STRING',
    'DEFAULT' => '0'
);
{% endhighlight %}

#### AJAX обработчик
Теперь, нужно добавить вывод этого параметра в шаблон. Для этого открываем файл (в шаблоне корзины) mutator.php. В этом файле собран массив с данными для вывода в представление. Обращение к этому файлу происходит при каждом запросе и массив отрабатывает на лету (AJAX). Файл фомирует результат для шаблонизатора

Вставляем код до формирование массива $totalData:


{% highlight ruby %}
// Считаем общее количество товаров в корзине.
$dbBasketItems = CSaleBasket::GetList(
	false,
	array("FUSER_ID" => CSaleBasket::GetBasketUserID(), "LID" => SITE_ID, "ORDER_ID" => "NULL"),
	false,
	false,
	array("ID","PRODUCT_ID","QUANTITY"));
while ($arItems=$dbBasketItems->Fetch())
{
	$arItems=CSaleBasket::GetByID($arItems["ID"]);
	$countItemsInCart+=$arItems['QUANTITY'];
}

$fixItemsInCart = $this->arParams['MAX_COUNT_ITEMS'];
if ($countItemsInCart > $fixItemsInCart) {
	$allcountItemsInCart = $fixItemsInCart;
	}

{% endhighlight %}

В конец массива добавляем ключ COUNT_ITEMS

{% highlight ruby %}
$totalData = array(
	'DISABLE_CHECKOUT' => (int)$result['ORDERABLE_BASKET_ITEMS_COUNT'] === 0,
	'PRICE' => $result['allSum'],
	'PRICE_FORMATED' => str_replace("руб.", "&#8381;", $result['allSum_FORMATED']),
	'PRICE_WITHOUT_DISCOUNT_FORMATED' => str_replace("руб.", "&#8381;", $result['PRICE_WITHOUT_DISCOUNT']),
	'CURRENCY' => $result['CURRENCY'],
	'COUNT_ITEMS' => $allcountItemsInCart
);

{% endhighlight %}

#### Выводим параметр пользователям
И осталось вывести и обработать этот параметр для пользователей. Открываем файл /local/templates/ШАБЛОН_САЙТА/components/bitrix/sale.basket.basket/js-templates/basket-total.php он отвечает за формирование и вывод низа корзины: кнопка перехода к оформлению.
Находим строчку, в которой прописана кнопка перехода к оформлению и заворачиваем в условие.

{% highlight ruby %}
<div class="basket__submit">


            {{#COUNT_ITEMS}}
            <div class="minimal_summ">
            <button disabled class="btn btn-lg">
                ОФОРМИТЬ ЗАКАЗ
          </div>
             <font color="red" size="2"><br> Максимальное количество пар обуви в заказе — <br>не более {{{COUNT_ITEMS}}}. Пожалуйста, внесите изменения <br>в заказ или оформите несколько заказов.</font>
            {{/COUNT_ITEMS}}
            {{^COUNT_ITEMS}}
                <button data-entity="basket-checkout-button" class="ok-button">
                        <?=Loc::getMessage('SBB_ORDER')?>
                </button>
            <button class="ok-button-secondary fastBasketOrder"><?=Loc::getMessage('SBB_FAST_ORDER')?></button>
            {{/COUNT_ITEMS}}

        </div>
{% endhighlight %}


Здесь проверяем: если из файла mutator.php пришло значение для COUNT_ITEMS изменяем кнопку оформить заказ на неактивную и выводим оповещение.


#### Максимальное количество товара на станице оформления заказа
Некоторые пользователи могут сохранить прямую ссылку и перейти на страницу оформления заказа. Что бы избежать этого, можно также проверять максимально доступное количество товара и отправлять таких пользователей обратно в корзину.


Открываем файл local/templates/ШАБЛОН_САЙТА/components/bitrix/sale.order.ajax/.parameters.php 

и у станавливаем код 

{% highlight ruby %}
$arTemplateParameters['MAX_COUNT_ITEMS_'] = array(
    'PARENT' => 'BASE',
    'NAME' => "максимальное количество товаров в корзине",
    'TYPE' => 'STRING',
    'DEFAULT' => '0'
);

{% endhighlight %}
в эту часть файла

{% highlight ruby %}
	$arTemplateParameters["YM_GOALS_SAVE_ORDER"] = array(
		"NAME" => GetMessage("YM_GOALS_SAVE_ORDER"),
		"TYPE" => "STRING",
		"DEFAULT" => "BX-order-save",
		"PARENT" => "ANALYTICS_SETTINGS"
	);
}

$arTemplateParameters['MAX_COUNT_ITEMS_'] = array(
    'PARENT' => 'BASE',
    'NAME' => "максимальное количество товаров в корзине",
    'TYPE' => 'STRING',
    'DEFAULT' => '0'
);

$arTemplateParameters['USE_ENHANCED_ECOMMERCE'] = array(
	'PARENT' => 'ANALYTICS_SETTINGS',
	'NAME' => GetMessage('USE_ENHANCED_ECOMMERCE'),
	'TYPE' => 'CHECKBOX',
	'REFRESH' => 'Y',
	'DEFAULT' => 'N'
);

if (isset($arCurrentValues['USE_ENHANCED_ECOMMERCE']) && $arCurrentValues['USE_ENHANCED_ECOMMERCE'] === 'Y')
{
	if (Loader::includeModule('catalog'))
	{
{% endhighlight %}


#### Добавим условие для редиректа пользователя обратно в корзину



открываем файл шаблона local/templates/ШАБЛОН_САЙТА/components/bitrix/sale.order.ajax/template.php у станавливаем код 

{% highlight ruby %}
  // Общее количество товаров в корзине.
  $dbBasketItems_ = CSaleBasket::GetList(
	false,
	array("FUSER_ID" => CSaleBasket::GetBasketUserID(), "LID" => SITE_ID, "ORDER_ID" => "NULL"),
	false,
	false,
	array("ID","PRODUCT_ID","QUANTITY"));
while ($arItems=$dbBasketItems_->Fetch())
{
	$arItems=CSaleBasket::GetByID($arItems["ID"]);
	$countItemsInCart_+=$arItems['QUANTITY'];
}

// Редирект в корзину если общее количество товаров в корзине более 4 пар..
if ($countItemsInCart_ > $arParams['MAX_COUNT_ITEMS_'] ) 
	{
	header('Location:' . '/personal/cart/');
	}
{% endhighlight %}

в начало файла

{% highlight ruby %}
<? if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) die();

use Bitrix\Main;
use Bitrix\Main\Localization\Loc;

/**
 * @var array $arParams
 * @var array $arResult
 * @var CMain $APPLICATION
 * @var CUser $USER
 * @var SaleOrderAjax $component
 * @var string $templateFolder
 */

  // Общее количество товаров в корзине.
  $dbBasketItems_ = CSaleBasket::GetList(
	false,
	array("FUSER_ID" => CSaleBasket::GetBasketUserID(), "LID" => SITE_ID, "ORDER_ID" => "NULL"),
	false,
	false,
	array("ID","PRODUCT_ID","QUANTITY"));
while ($arItems=$dbBasketItems_->Fetch())
{
	$arItems=CSaleBasket::GetByID($arItems["ID"]);
	$countItemsInCart_+=$arItems['QUANTITY'];
}

// Редирект в корзину если общее количество товаров в корзине более 4 пар..
if ($countItemsInCart_ > $arParams['MAX_COUNT_ITEMS_'] ) 
	{
	header('Location:' . '/personal/cart/');
	}

{% endhighlight %}
