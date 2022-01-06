# Области применения инъекций

Для людей, изучающих разные языки программирования, может быть неожиданным узнать, что в Nest почти все разделяется 
между входящими запросами. У нас есть пул соединений с базой данных, синглтонные сервисы с глобальным состоянием и т.д. 
Помните, что Node.js не следует многопоточной Stateless-модели запроса/ответа, в которой каждый запрос обрабатывается 
отдельным потоком. Следовательно, использование экземпляров синглтонов полностью **безопасно** для наших приложений.

Однако есть особые случаи, когда время жизни на основе запроса может быть желаемым поведением, например, кэширование 
по запросам в GraphQL-приложениях, отслеживание запросов и многопоточность. Области инъекций предоставляют механизм для 
получения желаемого поведения времени жизни провайдера.

## Область действия провайдера

Провайдер может иметь любую из следующих сфер действия:

<table>
  <tr>
    <td><code>DEFAULT</code></td>
    <td>
        Один экземпляр провайдера используется совместно со всем приложением. Время жизни экземпляра напрямую связано 
        с жизненным циклом приложения. После загрузки приложения все синглтонные провайдеры инстанцируются. По умолчанию 
        используется синглтонная область видимости.
    </td>
  </tr>
  <tr>
    <td><code>REQUEST</code></td>
    <td>
        Новый экземпляр провайдера создается исключительно для каждого входящего <strong>запроса</strong>. 
        После завершения обработки запроса экземпляр удаляется.
    </td>
  </tr>
  <tr>
    <td><code>TRANSIENT</code></td>
    <td>
        Переходные провайдеры не разделяются между потребителями. Каждый потребитель, инжектирующий переходного 
        провайдера, получает новый, выделенный экземпляр.
    </td>
  </tr>
</table>

> Использование области видимости singleton **рекомендуется** для большинства случаев использования. Совместное 
> использование провайдеров потребителями и запросами означает, что экземпляр может быть кэширован, а его инициализация 
> происходит только один раз, во время запуска приложения.

## Использование

Укажите область применения инъекции, передав свойство `scope` объекту опций декоратора `@Injectable()`:

```typescript
import { Injectable, Scope } from '@nestjs/common';
@Injectable({ scope: Scope.REQUEST })
export class CatsService {}
```

Аналогично, для [пользовательских провайдеров](/guide/fundamentals/custom-providers.html), установите свойство `scope` в развернутой 
форме регистрации провайдера:

```typescript
{
  provide: 'CACHE_MANAGER',
  useClass: CacheManager,
  scope: Scope.TRANSIENT,
}
```

> Импортируйте перечисление `Scope` из `@nestjs/common`.

> Шлюзы не должны использовать провайдеров, скопированных на запрос, поскольку они должны действовать как синглтоны. 
> Каждый шлюз инкапсулирует реальный сокет и не может быть инстанцирован несколько раз.

По умолчанию используется синглтонная область видимости, и ее не нужно объявлять. Если вы все же хотите объявить 
провайдера как singleton scoped, используйте значение `Scope.DEFAULT` для свойства `scope`.

## Область применения контроллера

Контроллеры также могут иметь область видимости, которая распространяется на все обработчики методов запроса, 
объявленные в этом контроллере. Как и область видимости провайдера, область видимости контроллера определяет время 
его жизни. Для контроллера, скопированного на запрос, новый экземпляр создается для каждого входящего запроса, и уничтожается, 
когда запрос завершает обработку.

Объявите область видимости контроллера с помощью свойства `scope` объекта `ControllerOptions`:

```typescript
@Controller({
  path: 'cats',
  scope: Scope.REQUEST,
})
export class CatsController {}
```

## Иерархия Scope

Сфера действия расширяется по цепочке инъекций. Контроллер, который зависит от провайдера, скопированного на запрос, 
сам будет скопирован на запрос.

Представьте себе следующий граф зависимостей: `CatsController <- CatsService <- CatsRepository`. Если `CatsService` 
является request-scoped (а остальные являются синглтонами по умолчанию), то `CatsController` станет request-scoped, 
поскольку он зависит от инжектированного сервиса. Хранилище `CatsRepository`, которое не является зависимым, останется 
singleton-scoped.

<demo-component></demo-component>

## Провайдер запросов

В приложении на основе HTTP-сервера (например, с использованием `@nestjs/platform-express` или `@nestjs/platform-fastify`) 
при использовании провайдеров с копированием запросов вы можете захотеть получить доступ к ссылке на исходный объект запроса. 
Вы можете сделать это, инжектируя объект `REQUEST`.

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';
@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(REQUEST) private request: Request) {}
}
```

Из-за различий в базовой платформе/протоколе, вы получаете доступ к входящему запросу немного по-разному для приложений 
Microservice или GraphQL. В приложениях [GraphQL](/guide/graphql/quick-start) вы вводите `CONTEXT` вместо `REQUEST`:

```typescript
import { Injectable, Scope, Inject } from '@nestjs/common';
import { CONTEXT } from '@nestjs/graphql';
@Injectable({ scope: Scope.REQUEST })
export class CatsService {
  constructor(@Inject(CONTEXT) private context) {}
}
```

Затем вы настраиваете значение `context` (в `GraphQLModule`), чтобы оно содержало `request` в качестве свойства.

## Производительность

Использование провайдеров с привязкой к запросу влияет на производительность приложения. Хотя Nest пытается кэшировать 
как можно больше метаданных, ему все равно придется создавать экземпляр вашего класса при каждом запросе. Следовательно, 
это замедлит среднее время отклика и общий результат бенчмаркинга. Если провайдер не должен быть скопирован на запрос, 
настоятельно рекомендуется использовать стандартную область видимости singleton.


