# Blkchnpdf

Модель классов:
 Описывает ключевые сущности: Пользователь, Магазин, Товар, Покупка, Возврат.
Диаграмма архитектуры:
 Включает модуль авторизации, модули управления, блокчейн и пользовательский
интерфейс.
9. Архитектура авторизации
Модуль авторизации реализован с использованием смарт-контракта, который хранит
данные пользователей и их роли. Доступ к функциям системы определяется ролью
пользователя.ыы
10. Документация функций смарт-контракта
 AddShop: Добавление нового магазина.
 AddItem: Добавление товара в магазин.
 BuyItem: Совершение покупки.
 RateItem: Оценка товара.
 RefundItemRequest: Запрос на возврат товара.
 RefundItemAccept/Reject: Обработка возвратов продавцо 


struct ReturnRequest {
    address payable customer;
    address payable shop;
    uint item_index;
    uint amount;
    uint price;
    bool isApproved;
    bool isProcessed;
}

mapping(address => ReturnRequest[]) returnRequests; 
mapping(address => uint[]) itemRatings; 


function RequestReturn(address payable shop_adr, uint item_index, uint amount) public {
    require(shops[shop_adr].exist, "Shop does not exist");
    require(item_index < items[shop_adr].length, "Item index out of range");
    require(amount > 0, "Amount must be greater than zero");

    bool found = false;
    for (uint i = 0; i < purchases[msg.sender].length; i++) {
        Purchase memory p = purchases[msg.sender][i];
        if (p.shop == shop_adr && p.item_index == item_index && p.amount >= amount) {
            found = true;
            break;
        }
    }
    require(found, "No matching purchase found");

    uint totalPrice = amount * items[shop_adr][item_index].price;

    ReturnRequest memory request;
    request.customer = payable(msg.sender);
    request.shop = shop_adr;
    request.item_index = item_index;
    request.amount = amount;
    request.price = totalPrice;
    request.isApproved = false;
    request.isProcessed = false;

    returnRequests[shop_adr].push(request);
}

function ApproveReturn(address customer, uint requestIndex) public OnlyEmployee(shops[msg.sender].shop_adr) {
    require(requestIndex < returnRequests[msg.sender].length, "Request index out of range");
    ReturnRequest storage request = returnRequests[msg.sender][requestIndex];
    require(!request.isProcessed, "Request already processed");
    require(request.customer == customer, "Invalid customer");

    request.isApproved = true;
    request.isProcessed = true;

    // Обновляем количество товаров
    items[request.shop][request.item_index].amount += request.amount;

    // Возврат средств покупателю
    (bool success, ) = request.customer.call{value: request.price}("");
    require(success, "Refund failed");
}

function GetItemRating(address shop_adr, uint item_index) public view returns (uint averageRating) {
    require(shops[shop_adr].exist, "Shop does not exist");
    require(item_index < items[shop_adr].length, "Item index out of range");

    uint totalRatings = itemRatings[shop_adr][item_index];
    uint totalCount = items[shop_adr][item_index].amount;

    return totalCount > 0 ? totalRatings / totalCount : 0;
}

function RateItem(address shop_adr, uint item_index, uint rating) public {
    require(shops[shop_adr].exist, "Shop does not exist");
    require(item_index < items[shop_adr].length, "Item index out of range");
    require(rating >= 1 && rating <= 5, "Rating must be between 1 and 5");

    bool found = false;
    for (uint i = 0; i < purchases[msg.sender].length; i++) {
        Purchase memory p = purchases[msg.sender][i];
        if (p.shop == shop_adr && p.item_index == item_index) {
            found = true;
            break;
        }
    }
    require(found, "No matching purchase found");

    itemRatings[shop_adr][item_index] += rating;
}

