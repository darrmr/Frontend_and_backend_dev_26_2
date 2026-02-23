|||
|---|---|
|ДИСЦИПЛИНА|Фронтенд и бэкенд разработка|
|ИНСТИТУТ|ИПТИП|
|КАФЕДРА|Индустриального программирования|
|ВИД УЧЕБНОГО МАТЕРИАЛА|Методические указания к практическим занятиям по дисциплине|
|ПРЕПОДАВАТЕЛЬ|Загородних Николай Анатольевич<br>Краснослободцева Дарья Борисовна|
|СЕМЕСТР|4 семестр, 2025/2026 уч. год|

# Практическое занятие 5
## Расширенный REST API 

Подробнее рассмотрим REST API, а также нструментарий для его проектирования, документирования, тестирования и развертывания - Swagger. Решение практического задания осуществляется внутри соответствующей рабочей тетради, расположенной в СДО.

### Что такое Swagger и зачем он нужен?

Swagger (или OpenAPI) - это набор инструментов для проектирования, документирования и тестирования REST API. Вместо того чтобы писать документацию вручную в Wiki или Word-документах, разработчики описывают API в структурированном виде (YAML или JSON), а Swagger автоматически превращает это описание в красивый и интерактивный веб-интерфейс.

**Преимущества использования Swagger:**
1.  Интерактивность: прямо в браузере можно отправлять запросы к API и видеть ответы.
2.  Актуальность: документация всегда соответствует коду, если генерируется из аннотаций.
3.  Удобство: понятно не только разработчикам, но и тестировщикам, аналитикам и менеджерам.
4.  Стандартизация: OpenAPI стал индустриальным стандартом описания API.

В рамках практического занятия мы возьмем готовое приложение интернет-магазина (клиент на React и сервер на Express) и добавим к нему автоматическую генерацию документации Swagger на серверной части.

### Подготовка бэкенда

В качестве основы возьмем сервер на Express из предыдущего занятия, но адаптируем его под интернет-магазин. Мы будем использовать библиотеку `swagger-jsdoc` для генерации спецификации из JSDoc-комментариев и `swagger-ui-express` для отображения интерфейса.

#### Установка зависимостей

Перейдите в папку с вашим backend-проектом и установите необходимые пакеты:

```bash
npm i express nanoid cors
npm i -D swagger-jsdoc swagger-ui-express
```

#### Создание базового сервера с Swagger

Создадим файл `app.js`, в котором будет описан наш API интернет-магазина с полной документацией.

```js
const express = require('express');
const { nanoid } = require('nanoid');
const cors = require('cors');

// Подключаем Swagger
const swaggerJsdoc = require('swagger-jsdoc');
const swaggerUi = require('swagger-ui-express');

const app = express();
const port = 3000;

// Настройка CORS (аналогично проекту, реализованному в ромках Практической работы №4)
app.use(cors({
    origin: "http://localhost:3001",
    methods: ["GET", "POST", "PATCH", "DELETE"],
    allowedHeaders: ["Content-Type", "Authorization"],
}));

// Middleware для парсинга JSON
app.use(express.json());

// Middleware для логирования запросов
app.use((req, res, next) => {
    res.on('finish', () => {
        console.log(`[${new Date().toISOString()}] [${req.method}] ${res.statusCode} ${req.path}`);
    });
    next();
});

// Наши данные
let products = [
    { id: nanoid(6), name: 'Ноутбук ASUS', category: 'Электроника', description: 'Мощный ноутбук для работы и игр', price: 75000, stock: 5 },
    { id: nanoid(6), name: 'Наушники Sony', category: 'Аксессуары', description: 'Беспроводные наушники с шумоподавлением', price: 12000, stock: 12 },
    { id: nanoid(6), name: 'Кофеварка DeLonghi', category: 'Для дома', description: 'Автоматическая капсульная кофеварка', price: 8500, stock: 3 },
];

// Swagger definition
// Описание основного API
const swaggerOptions = {
    definition: {
        openapi: '3.0.0',
        info: {
            title: 'API интернет-магазина',
            version: '1.0.0',
            description: 'Простое API для управления товарами в интернет-магазине',
        },
        servers: [
            {
                url: `http://localhost:${port}`,
                description: 'Локальный сервер',
            },
        ],
    },
    // Путь к файлам, в которых мы будем писать JSDoc-комментарии (наш текущий файл)
    apis: ['./app.js'],
};

const swaggerSpec = swaggerJsdoc(swaggerOptions);

// Подключаем Swagger UI по адресу /api-docs
app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec));

/**
 * @swagger
 * components:
 *   schemas:
 *     Product:
 *       type: object
 *       required:
 *         - name
 *         - price
 *       properties:
 *         id:
 *           type: string
 *           description: Автоматически сгенерированный уникальный ID товара
 *         name:
 *           type: string
 *           description: Название товара
 *         category:
 *           type: string
 *           description: Категория товара
 *         description:
 *           type: string
 *           description: Подробное описание товара
 *         price:
 *           type: number
 *           description: Цена товара в рублях
 *         stock:
 *           type: integer
 *           description: Количество товара на складе
 *       example:
 *         id: "abc123"
 *         name: "Ноутбук ASUS"
 *         category: "Электроника"
 *         description: "Мощный ноутбук для работы и игр"
 *         price: 75000
 *         stock: 5
 */

// Функция-помощник для поиска товара
function findProductOr404(id, res) {
    const product = products.find(p => p.id == id);
    if (!product) {
        res.status(404).json({ error: "Product not found" });
        return null;
    }
    return product;
}

/**
 * @swagger
 * /api/products:
 *   post:
 *     summary: Создает новый товар
 *     tags: [Products]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required:
 *               - name
 *               - price
 *             properties:
 *               name:
 *                 type: string
 *               category:
 *                 type: string
 *               description:
 *                 type: string
 *               price:
 *                 type: number
 *               stock:
 *                 type: integer
 *     responses:
 *       201:
 *         description: Товар успешно создан
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Product'
 *       400:
 *         description: Ошибка в теле запроса
 */
app.post("/api/products", (req, res) => {
    const { name, category, description, price, stock } = req.body;

    if (!name || price === undefined) {
        return res.status(400).json({ error: "Name and price are required" });
    }

    const newProduct = {
        id: nanoid(6),
        name: name.trim(),
        category: category?.trim() || "Без категории",
        description: description?.trim() || "",
        price: Number(price),
        stock: stock !== undefined ? Number(stock) : 0,
    };

    products.push(newProduct);
    res.status(201).json(newProduct);
});

/**
 * @swagger
 * /api/products:
 *   get:
 *     summary: Возвращает список всех товаров
 *     tags: [Products]
 *     responses:
 *       200:
 *         description: Список товаров
 *         content:
 *           application/json:
 *             schema:
 *               type: array
 *               items:
 *                 $ref: '#/components/schemas/Product'
 */
app.get("/api/products", (req, res) => {
    res.json(products);
});

/**
 * @swagger
 * /api/products/{id}:
 *   get:
 *     summary: Получает товар по ID
 *     tags: [Products]
 *     parameters:
 *       - in: path
 *         name: id
 *         schema:
 *           type: string
 *         required: true
 *         description: ID товара
 *     responses:
 *       200:
 *         description: Данные товара
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Product'
 *       404:
 *         description: Товар не найден
 */
app.get("/api/products/:id", (req, res) => {
    const product = findProductOr404(req.params.id, res);
    if (!product) return;
    res.json(product);
});

/**
 * @swagger
 * /api/products/{id}:
 *   patch:
 *     summary: Обновляет данные товара
 *     tags: [Products]
 *     parameters:
 *       - in: path
 *         name: id
 *         schema:
 *           type: string
 *         required: true
 *         description: ID товара
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             properties:
 *               name:
 *                 type: string
 *               category:
 *                 type: string
 *               description:
 *                 type: string
 *               price:
 *                 type: number
 *               stock:
 *                 type: integer
 *     responses:
 *       200:
 *         description: Обновленный товар
 *         content:
 *           application/json:
 *             schema:
 *               $ref: '#/components/schemas/Product'
 *       400:
 *         description: Нет данных для обновления
 *       404:
 *         description: Товар не найден
 */
app.patch("/api/products/:id", (req, res) => {
    const product = findProductOr404(req.params.id, res);
    if (!product) return;

    if (Object.keys(req.body).length === 0) {
        return res.status(400).json({ error: "Nothing to update" });
    }

    const { name, category, description, price, stock } = req.body;

    if (name !== undefined) product.name = name.trim();
    if (category !== undefined) product.category = category.trim();
    if (description !== undefined) product.description = description.trim();
    if (price !== undefined) product.price = Number(price);
    if (stock !== undefined) product.stock = Number(stock);

    res.json(product);
});

/**
 * @swagger
 * /api/products/{id}:
 *   delete:
 *     summary: Удаляет товар
 *     tags: [Products]
 *     parameters:
 *       - in: path
 *         name: id
 *         schema:
 *           type: string
 *         required: true
 *         description: ID товара
 *     responses:
 *       204:
 *         description: Товар успешно удален (нет тела ответа)
 *       404:
 *         description: Товар не найден
 */
app.delete("/api/products/:id", (req, res) => {
    const id = req.params.id;

    const exists = products.some((p) => p.id === id);
    if (!exists) return res.status(404).json({ error: "Product not found" });

    products = products.filter((p) => p.id !== id);
    res.status(204).send();
});

// 404 для всех остальных маршрутов
app.use((req, res) => {
    res.status(404).json({ error: "Not found" });
});

// Глобальный обработчик ошибок
app.use((err, req, res, next) => {
    console.error("Unhandled error:", err);
    res.status(500).json({ error: "Internal server error" });
});

// Запуск сервера
app.listen(port, () => {
    console.log(`Сервер запущен на http://localhost:${port}`);
    console.log(`Swagger UI доступен по адресу http://localhost:${port}/api-docs`);
});
```

### Документирование API с помощью JSDoc

Обратите внимание на комментарии, начинающиеся с `/** @swagger */`. Это и есть JSDoc-аннотации, которые `swagger-jsdoc` превратит в спецификацию.

1.  **Описание схемы (`components/schemas/Product`):**
    Мы описали, как выглядит объект "Товар". Какие поля у него есть, какие из них обязательные, и привели пример. Теперь в других частях документации мы можем ссылаться на эту схему через `$ref: '#/components/schemas/Product'`.

2.  **Описание эндпоинтов:**
    Для каждого HTTP-метода (`POST /api/products`, `GET /api/products`) мы добавили:
    *   `summary` - краткое описание.
    *   `tags`- группировка методов в интерфейсе Swagger.
    *   `parameters` - описание параметров пути (например, `id`).
    *   `requestBody`- описание тела запроса (для POST и PATCH).
    *   `responses`- описание возможных ответов от сервера (коды 200, 201, 404 и т.д.) и примеры данных, которые придут.

### Запуск и просмотр документации

1.  Запустите сервер командой `node app.js`.
2.  Откройте браузер и перейдите по адресу `http://localhost:3000/api-docs`.

Вы увидите интерактивную документацию:

......................Тут картиночка

Теперь вы можете изучить структуру API и развернуть любой запрос (например, `GET /api/products`). Также можете нажать кнопку "Try it out", чтобы отправить реальный запрос к вашему API и увидеть ответ прямо в браузере.

### Работа с фронтендом

Поскольку мы поменяли структуру данных (был `user`, стал `product`) и эндпоинты (было `/api/users`, стало `/api/products`), фронтенд из предыдущей работы нужно немного доработать.

Основные изменения в `src/api/index.js`:

```js
import axios from "axios";

const apiClient = axios.create({
    baseURL: "http://localhost:3000/api", // Базовый URL тот же
    headers: {
        "Content-Type": "application/json",
        "accept": "application/json",
    }
});

export const api = {
    // Вместо createUser
    createProduct: async (product) => {
        let response = await apiClient.post("/products", product);
        return response.data;
    },
    // Вместо getUsers
    getProducts: async () => {
        let response = await apiClient.get("/products");
        return response.data;
    },
    // Вместо updateUser
    updateProduct: async (id, product) => {
        let response = await apiClient.patch(`/products/${id}`, product);
        return response.data;
    },
    // Вместо deleteUser
    deleteProduct: async (id) => {
        let response = await apiClient.delete(`/products/${id}`);
        return response.data;
    }
}
```

Соответственно, в компонентах (`ProductsPage.jsx`, `ProductItem.jsx`, `ProductModal.jsx`) нужно заменить:
*   `users` на `products`.
*   `name` на `name` (оставляем, это название товара).
*   Добавить поля `category`, `description`, `price`, `stock` в форму и карточку товара.

### Практическое задание

Необходимо доработать рассмотренный пример:
1.  Создайте сервер на Express, который управляет товарами интернет-магазина.
2.  Подключите к нему `swagger-jsdoc` и `swagger-ui-express`.
3.  Опишите с помощью JSDoc-аннотаций схему товара (`Product`) и все CRUD-операции (`GET`, `POST`, `GET/:id`, `PATCH/:id`, `DELETE`).
4.  Убедитесь, что документация доступна по адресу `/api-docs` и работает в интерактивном режиме (можно отправить тестовый запрос).
5.  (Опционально) Адаптируйте фронтенд на React из 4-го занятия для работы с товарами и их отображения.

### Формат отчета

В качестве ответа на задание необходимо прикрепить ссылку на репозиторий с реализованной практикой. Ссылка подгружается в соответствующий раздел СДО: Задания для самостоятельной работы -> Практические занятия 5-6.

