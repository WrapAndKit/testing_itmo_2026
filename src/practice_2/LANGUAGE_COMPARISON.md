## Подбор стека по философии

| Язык | Фреймворк API | Тестовый фреймворк | Библиотека моков | Даты |
|------|--------------|---------------------|------------------|------|
| Python | FastAPI | pytest | `unittest.mock` (встроенная) | `datetime` + `zoneinfo` |
| TypeScript | NestJS | Jest | встроены в Jest (`jest.fn()`) | Luxon |
| C# / .NET 8 | ASP.NET Core | xUnit | Moq | `DateOnly` + `DateTimeOffset` |

Все три стека имеют:
- **DI-контейнер**, позволяющий подменять зависимости в тестах
- **Валидацию DTO** (Pydantic / class-validator / DataAnnotations)
- **In-memory API-тесты** без поднятия сервера (TestClient / NestJS TestingModule / WebApplicationFactory)

---

## Таблица ключевых аналогий

### Моки (Пара 3)

| Действие | Python (MagicMock) | TypeScript (Jest) | C# (Moq) |
|----------|--------------------|--------------------|-----------|
| Создать мок | `MagicMock(spec=IFoo)` | `jest.Mocked<IFoo>` + `jest.fn()` | `new Mock<IFoo>()` |
| Простое значение | `m.method.return_value = 42` | `m.method.mockReturnValue(42)` | `m.Setup(x => x.Method()).Returns(42)` |
| Async значение | (то же, но `await`) | `m.method.mockResolvedValue(42)` | `m.Setup(...).ReturnsAsync(42)` |
| Функция-имплементация | `m.method.side_effect = fn` | `m.method.mockImplementation(fn)` | `m.Setup(...).ReturnsAsync((args) => fn(args))` |
| Исключение | `m.method.side_effect = Error(...)` | `m.method.mockRejectedValue(new Error())` | `m.Setup(...).ThrowsAsync(new Exception())` |
| Последовательность | `m.method.side_effect = [a, b, c]` | `m.method.mockResolvedValueOnce(a).mockResolvedValueOnce(b)` | `m.SetupSequence(...).ReturnsAsync(a).ReturnsAsync(b)` |
| Проверка одного вызова | `m.method.assert_called_once_with(x)` | `expect(m.method).toHaveBeenCalledWith(x)` + `.toHaveBeenCalledTimes(1)` | `m.Verify(x => x.Method(arg), Times.Once)` |
| Проверка отсутствия вызова | `m.method.assert_not_called()` | `expect(m.method).not.toHaveBeenCalled()` | `m.Verify(..., Times.Never)` |
| N-ый вызов | `m.method.call_args_list[i]` | `m.method.mock.calls[i]` | `m.Invocations[i].Arguments` |

### API-тесты (Пара 4)

| Действие | FastAPI | NestJS | ASP.NET Core |
|----------|---------|--------|--------------|
| Создать тестовый клиент | `TestClient(app)` | `Test.createTestingModule({...}).compile()` + `supertest(app.getHttpServer())` | `new WebApplicationFactory<Program>().CreateClient()` |
| GET-запрос | `client.get("/path")` | `request(server).get("/path")` | `await client.GetAsync("/path")` |
| POST с JSON | `client.post("/path", json={...})` | `request(server).post("/path").send({...})` | `await client.PostAsJsonAsync("/path", obj)` |
| Статус | `response.status_code` | `response.status` | `response.StatusCode` |
| JSON-тело | `response.json()` | `response.body` | `await response.Content.ReadFromJsonAsync<T>()` |
| Подмена зависимости | `app.dependency_overrides[fn] = ...` | `.overrideProvider(Token).useValue(...)` | `ConfigureTestServices(s => { s.Remove(orig); s.Add(new) })` |
| Параметризация | `@pytest.mark.parametrize(...)` | `it.each([...])` | `[Theory] + [InlineData(...)]` |

---

## Пример «одинакового» теста на трёх языках

**Задача:** проверить, что `SettlementService.calculateSettlement` корректно использует курс 100 для расчёта `quoteAmount = baseAmount × rate`.

### Python + pytest + unittest.mock

```python
def test_basic_settlement(moex):
    # Arrange
    provider = MagicMock(spec=ExchangeRateProvider)
    provider.get_rate.return_value = 100.0
    service = SettlementService(provider, moex)

    # Act
    result = service.calculate_settlement(
        date(2024, 11, 1), 1000.0, "USD", "RUB"
    )

    # Assert
    assert result.applied_rate == 100.0
    assert result.quote_amount == 100_000.0
    assert result.settlement_date == date(2024, 11, 6)

    provider.get_rate.assert_called_once_with(
        base="USD", quote="RUB", on_date=date(2024, 11, 1)
    )
```

### TypeScript + Jest

```typescript
it('basic settlement with rate 100', async () => {
  // Arrange
  const provider: jest.Mocked<ExchangeRateProvider> = { getRate: jest.fn() };
  provider.getRate.mockResolvedValue(100.0);
  const service = new SettlementService(provider, MOEX);

  // Act
  const result = await service.calculateSettlement(
    DateTime.fromISO('2024-11-01', { zone: 'utc' }),
    1000.0,
    'USD',
    'RUB',
  );

  // Assert
  expect(result.appliedRate).toBe(100.0);
  expect(result.quoteAmount).toBe(100_000);
  expect(result.settlementDate.toISODate()).toBe('2024-11-06');

  expect(provider.getRate).toHaveBeenCalledTimes(1);
  expect(provider.getRate).toHaveBeenCalledWith(
    'USD', 'RUB', expect.anything(),
  );
});
```

### C# + xUnit + Moq

```csharp
[Fact]
public async Task Basic_settlement_with_rate_100()
{
    // Arrange
    var mock = new Mock<IExchangeRateProvider>();
    mock.Setup(x => x.GetRateAsync(
            It.IsAny<string>(), It.IsAny<string>(), It.IsAny<DateOnly>()))
        .ReturnsAsync(100m);

    var service = new SettlementService(mock.Object, Exchanges.Moex);

    // Act
    var result = await service.CalculateSettlementAsync(
        DateOnly.Parse("2024-11-01"), 1000m, "USD", "RUB");

    // Assert
    result.AppliedRate.Should().Be(100m);
    result.QuoteAmount.Should().Be(100_000m);
    result.SettlementDate.Should().Be(DateOnly.Parse("2024-11-06"));

    mock.Verify(
        x => x.GetRateAsync("USD", "RUB", DateOnly.Parse("2024-11-01")),
        Times.Once);
}
```

---

## Когда выбирать какой стек

### Python + FastAPI
- **Когда:** дата-сайенс, ML, скрипты, стартапы
- **Плюсы:** очень лаконичный, быстрый старт, динамика помогает в тестах
- **Минусы:** ошибки типов ловятся только в рантайме; мок с опечатками может молча пройти

### TypeScript + NestJS
- **Когда:** веб-команда, общий стек с фронтендом, Node.js-инфраструктура
- **Плюсы:** строгая типизация, большая экосистема, async по умолчанию
- **Минусы:** верботный код, сложная настройка (tsconfig, jest config); моки требуют `jest.Mocked<T>` для типобезопасности

### C# + ASP.NET Core
- **Когда:** корпоративная разработка, финтех, банки
- **Плюсы:** самая строгая типизация, мощная IDE-поддержка, производительность
- **Минусы:** verbose code, медленнее писать тесты, больше boilerplate

---
