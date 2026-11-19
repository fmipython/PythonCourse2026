# Домашно 2

Сестрата на Декстър, Диди, успя да влезе в лабораторията на брат си и да обърка кода на агента, който той беше написал. Декстър отново се обръща към вас за помощ за да създадете още по-добър агент от изтрития.

## Функционални промени

Досега нямахме възможност да си говорим с нашия агент. Време е да добавим такава възможност. За целта обаче, трябва да се отървем от XML-а, и да го заменим с JSON.

Имплементацията на класът `LLM` остава извън това, което трябва да имплементирате. В решението си оставете следния stub (ще го използвате в type hint-овете на агента):

```python
from collections.abc import Generator
from typing import Any


class LLM:
    def generate(self, user_message: dict[str, str]) -> Generator[dict[str, Any], None, None]:
        raise NotImplementedError
```

Обект от класа все още ще се подава като аргумент на нашия агент. Методът `run` на агента вече приема потребителското съобщение като низ - `run(user_message: str)`. В него трябва да го форматирате като потребителско съобщение (виж по-долу) и да го подадете на метода `generate` на `LLM`. Като резултат ще получите поредица от съобщения на модела (отново като генератор).

Първо ще дефинираме потребителското съобщение по следния начин:

```json
{
    "content": <response_text>,
    "role": "user"
}
```

А това от модела, по следния начин:

```json
{
    "model": <name_of_model>,
    "content": <response_text>,
    "role": "assistant",
    "finish_reason": null
}
```

Извикване на tool се случва по следния начин

```json
{
    "model": <name_of_model>,
    "content": "",
    "role": "assistant",
    "tool_calls": [
        {
            "tool_name": <tool_name_1>,
            "arguments": [
                "arg1", "arg2", ...
            ]
        },
        {
            "tool_name": <tool_name_2>,
            "arguments": [
                "arg3", "arg4", ...
            ]
        }
    ],
    "finish_reason": "tool"
}
```

Вече ще поддържаме изпълнението на няколко tool-а в рамките на едно съобщение. За всеки tool агентът връща (`yield`) низ "Tool result: " и резултата от tool-а (например "Tool result: 9"). Резултатите **не** се изпращат обратно към модела.

За край на отговор ще се връща:

```json
{
    "model": <name_of_model>,
    "content": null,
    "role": "assistant",
    "finish_reason": "stop",
    "usage": {
        "completion_tokens": <usage>
    }
}
```

Логиката за изчисляване на използваните token-и вече е извън агента, ще бъде връщана от модела.

`run` е генератор и връща (`yield`):

- За съобщение от модела с `finish_reason` `null` - неговото `content`.
- За край на отговор (`finish_reason` `"stop"`) - наредена двойка `("stop", <usage>)`, където `<usage>` е стойността на `completion_tokens`. След това агентът спира да обработва съобщенията от текущото извикване на `generate` - всички следващи съобщения от модела се игнорират (и не се записват в историята).
- За всеки tool call - "Tool result: ...", "Memory written" или "Memory loaded: ..." (виж по-долу).
- При грешка от tool (виж "Изключения") - текста на грешката.

Друга важна промяна - достъпа до паметта ще е през tool call. Достъпването ще става през `read_memory`, а писането е през `write_memory`. Паметта остава в агента, като вече е речник с низове за ключ и стойност.
Tool-овете `read_memory` и `write_memory` се обработват директно от агента, а не през `ToolRegistry` - не е нужно да бъдат регистрирани, и имат предимство пред tool-ове със същото име в `ToolRegistry`.
`write_memory` приема ключ като първи аргумент, а стойността е останалите аргументи, слепени с интервал (например `["name", "Dexter", "Lab"]` записва `"Dexter Lab"` под ключ `"name"`). След записване агентът връща (`yield`) низа "Memory written".
При прочитане на паметта (`read_memory` приема ключа като единствен аргумент), агентът връща (`yield`) низ "Memory loaded: " и съдържанието на паметта за дадения ключ. Ако ключът липсва в паметта, вместо съдържанието се използва "Memory not found" (т.е. низът е "Memory loaded: Memory not found"). Както и при останалите tool-ове, нищо от това не се изпраща към модела.

Към агента вкарваме метод `read_memory()`, който връща копие на паметта на модела.

Ще запазваме всички съобщения в агента - потребителските, всички съобщения от модела, както и грешките от tool-ове (виж "Изключения"). Можем да достъпим историята през метод `get_history()`, който връща копие на историята.

Промени по резултата от `read_memory()` или `get_history()` не трябва да променят паметта или историята на агента.

## Type hints

Допишете type hints в сигнатурите на всички методи. Няколко бележки:

- Аргументите на tool винаги са низове
- Изпълнението на tool връща низ, булева променлива, число (цяло или с плаваща запетая) или `None`

## Изключения

Ще дефинираме следните изключения:

- `UnknownToolError` - грешка, когато потърсим tool в `ToolRegistry`, който не съществува. Хвърля се с "{tool_name} is not found"
- `ToolExecutionError` - хвърля се с "{tool_name} raised an error.", ако някой tool хвърли грешка по време на изпълнението си

Агентът прихваща тези изключения и връща (`yield`) текста на грешката. Грешката се записва в историята като съобщение:

```json
{
    "content": <error_text>,
    "role": "error"
}
```

Грешката **не** се изпраща към модела, а агентът продължава със следващия tool call (ако има такъв).

## Пример

Нека моделът отговаря със следните съобщения:

```python
class ScriptedLLM(LLM):
    def generate(self, user_message):
        yield {"model": "dexter-1", "content": "Let me compute that.", "role": "assistant", "finish_reason": None}
        yield {
            "model": "dexter-1",
            "content": "",
            "role": "assistant",
            "tool_calls": [
                {"tool_name": "add", "arguments": ["2", "7"]},
                {"tool_name": "write_memory", "arguments": ["name", "Dexter", "Lab"]},
                {"tool_name": "read_memory", "arguments": ["name"]},
                {"tool_name": "read_memory", "arguments": ["age"]},
                {"tool_name": "multiply", "arguments": ["2", "3"]},
            ],
            "finish_reason": "tool",
        }
        yield {
            "model": "dexter-1",
            "content": None,
            "role": "assistant",
            "finish_reason": "stop",
            "usage": {"completion_tokens": 42},
        }
        yield {"model": "dexter-1", "content": "Ignored", "role": "assistant", "finish_reason": None}
```

```python
registry = ToolRegistry()
registry.register("add", add)

agent = Agent(ScriptedLLM(), registry)

for result in agent.run("What is 2 + 7?"):
    print(result)
```

Изход:

```
Let me compute that.
Tool result: 9
Memory written
Memory loaded: Dexter Lab
Memory loaded: Memory not found
multiply is not found
('stop', 42)
```

Последното съобщение (`"Ignored"`) не се връща, защото е след края на отговора.

```python
>>> agent.read_memory()
{'name': 'Dexter Lab'}
>>> for message in agent.get_history():
...     print(message)
{'content': 'What is 2 + 7?', 'role': 'user'}
{'model': 'dexter-1', 'content': 'Let me compute that.', 'role': 'assistant', 'finish_reason': None}
{'model': 'dexter-1', 'content': '', 'role': 'assistant', 'tool_calls': [...], 'finish_reason': 'tool'}
{'content': 'multiply is not found', 'role': 'error'}
{'model': 'dexter-1', 'content': None, 'role': 'assistant', 'finish_reason': 'stop', 'usage': {'completion_tokens': 42}}
```

Обърнете внимание, че грешката от tool-а е в историята **след** съобщението с tool call-овете, а последното съобщение (`"Ignored"`) липсва.

## Всичко останало

Всичко, което не е описано изрично тук, остава както в Домашно 1. Изключение е декораторът `observe` - той вече не е нужен.
