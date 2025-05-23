Ниже приведена подробная документация для скрипта **lateralus**. Обратите внимание, что данный скрипт был создан с использованием возможностей Copilot и разработан с учетом современных требований биоинформатики. Документация описывает цель, требования, порядок запуска, внутреннюю логику работы, используемые алгоритмы (включая вычисление идентичности) и форматы входных/выходных данных.

---

# Документация для скрипта **lateralus**

## 1. Обзор

**lateralus** — это скрипт на Python, предназначенный для автоматизированного улучшения аннотаций генома. Он сравнивает два файла в формате GenBank (целевой и референсный) и обновляет CDS‑записи (аннотации генов) в целевом файле на основе данных из референсного файла. Помимо обновленного GenBank‑файла, скрипт создает лог изменений в формате CSV, а при наличии библиотеки Pandas — также в формате Excel.

> **Примечание:**  
> Скрипт был создан с использованием возможностей Copilot, что позволило учесть современные подходы и реализовать оптимизированное решение для задачи автоматизации биоинформатических процессов.

---

## 2. Цель

Основные задачи скрипта:

- **Сравнение аннотаций генов:**  
  Для каждой CDS целевого файла производится поиск соответствующей записи в референсном файле. Сравнение выполняется на основе белковых последовательностей (поле `translation`) с использованием глобального выравнивания.

- **Редактирование аннотаций:**  
  - Если в целевой записи поле `gene` отсутствует или совпадает с `locus_tag` (что часто свидетельствует о базовой или неуточненной аннотации), а в найденной референсной записи поле `gene` задано корректно, то значение `gene` обновляется.
  - Если в поле `product` целевой записи еще не содержится обновленное имя гена, оно также заменяется на значение из референсного файла.

- **Логирование изменений:**  
  Детально фиксируются исходные значения, внесенные изменения и флаги (Да/Нет) для каждого поля (`gene` и `product`). Результаты записываются в лог в формате CSV, а при наличии Pandas — также в формате Excel.

- **Параллельная обработка:**  
  Скрипт использует параллельное выполнение (через `concurrent.futures.ProcessPoolExecutor`) для ускорения обработки большого объема CDS.

---

## 3. Требования

Для работы скрипта **lateralus** необходимо:

- **Python 3.7+**  
  Рекомендуется использовать последнюю стабильную версию Python.

- **Библиотека Biopython**  
  Используется для чтения, обработки и записи файлов формата GenBank.  
  _Установка:_  
  ```bash
  pip install biopython
  ```

- **Библиотека Pandas (опционально)**  
  Применяется для экспорта лога в формате Excel.  
  _Установка:_  
  ```bash
  pip install pandas openpyxl
  ```

- **Модуль concurrent.futures**  
  Встроен в стандартную библиотеку Python и используется для организации параллельной обработки.

---

## 4. Входные данные и запуск

Скрипт принимает два аргумента командной строки:

1. **Целевой файл GenBank** (например, `target.gb` или `target.gbk`) — аннотации которого будут обновлены.
2. **Референсный файл GenBank** (например, `reference.gb` или `reference.gbk`) — эталонный файл, откуда будут браться корректные аннотации.

Пример запуска:
```bash
python lateralus.py target.gb reference.gb
```
Каждый из входных файлов должен содержать одну запись генома с множеством аннотированных CDS‑функций.

---

## 5. Выходные данные

После выполнения скрипта создаются следующие файлы:

- **edited.gb**  
  Обновленный файл GenBank с переработанными аннотациями CDS.

- **results.csv**  
  Лог изменений с колонками:
  - **Целевой до изменений:** исходные значения полей `gene` и `product`
  - **Изменения из референсного:** описание внесённых изменений (например, `gene: old -> new`)
  - **Изменено название гена:** Да/Нет
  - **Изменено название продукта:** Да/Нет

- **results.xlsx**  
  (При наличии библиотеки Pandas) аналогичный лог изменений в формате Excel.

В консоль выводятся статистические данные:
- Процент CDS‑записей, где `gene` совпадает с `locus_tag` (до и после обработки).

---

## 6. Внутренняя логика работы

### 6.1 Чтение и подготовка данных

- **Загрузка файлов:**  
  С использованием модуля `SeqIO` из Biopython считываются входные файлы GenBank.

- **Подсчет статистики:**  
  Функция `compute_percentage(record)` проходит по всем CDS‑записям в файле и вычисляет процент случаев, когда значение поля `gene` совпадает с `locus_tag`. Это позволяет оценить исходное качество аннотации.

- **Извлечение информации о CDS:**  
  Из каждого файла извлекаются записи типа CDS, из которых сохраняются следующие поля:
  - `locus_tag` (сохраняется как `locus`)
  - `gene` (если отсутствует, приравнивается к `locus_tag`)
  - `product`
  - `translation` – белковая последовательность  
  Для целевого файла дополнительно сохраняется глобальный индекс записи (ключ `global_index`), что обеспечивает корректное обновление исходного объекта `target_record.features`.

### 6.2 Сравнение белковых последовательностей и вычисление идентичности

- **Глобальное выравнивание:**  
  Для сравнения белковых последовательностей применяется класс `PairwiseAligner` из Biopython с режимом `global`.

- **Вычисление идентичности:**  
  Идентичность вычисляется по следующей формуле:
  
  \[
  \text{Идентичность} = \frac{\text{score}}{\max(\text{len(target\_seq)},\, \text{len(ref\_seq)})}
  \]
  
  где:
  - **score** — оптимальный счет глобального выравнивания, вычисляемый методом `score()`,
  - **len(target_seq)** и **len(ref_seq)** — длины сравниваемых белковых последовательностей.
  
  Если полученная идентичность составляет не менее 70% (0.7), CDS целевого файла считается подходящей для обновления аннотации.

### 6.3 Редактирование аннотации

- **Обновление поля `gene`:**  
  Если в целевой записи:
  - Значение `gene` отсутствует или равно `locus_tag`,
  - А в соответствующей записи референсного файла поле `gene` задано корректно и отличается от `locus_tag` референсного файла,  
  то значение поля `gene` обновляется на значение из референсной записи.

- **Обновление поля `product`:**  
  Если в поле `product` целевой CDS не содержится уже обновленное имя гена, оно заменяется на значение из референсной CDS, если оно задано.

### 6.4 Параллельная обработка

Обработка каждой CDS производится параллельно с использованием класса `ProcessPoolExecutor` из модуля `concurrent.futures`. Результаты сохраняются в словарь с ключом, равным глобальному индексу записи, что позволяет корректно обновлять исходный объект `target_record.features`.

### 6.5 Логирование и сохранение результатов

- **Логирование:**  
  Для каждой CDS формируется строка лога, в которой фиксируются:
  - Исходные значения полей `gene` и `product`,
  - Описание внесенных изменений,
  - Флаги «Изменено название гена» и «Изменено название продукта» (Да/Нет).

- **Сохранение файлов:**  
  - Обновленный GenBank‑файл сохраняется под именем `edited.gb`.
  - Лог изменений записывается в файл `results.csv`, а при наличии Pandas — также в файл `results.xlsx`.

- **Консольный вывод:**  
  После завершения обработки повторно вычисляется статистика аннотаций (процент совпадения `gene` с `locus_tag`) для обновленного файла и выводится в консоль.

---

## 7. Использование скрипта

### Подготовка окружения

Убедитесь, что выполнена установка необходимых библиотек:
```bash
pip install biopython pandas openpyxl
```

### Запуск

Запустите скрипт, передав пути к целевому и референсному файлам:
```bash
python lateralus.py target.gb reference.gb
```

### Результаты

После выполнения скрипта будут созданы следующие файлы:
- **edited.gb:** обновленный GenBank‑файл с переработанными аннотациями.
- **results.csv:** лог изменений в формате CSV.
- **results.xlsx:** (при наличии Pandas) лог изменений в формате Excel.  
В консоль будут выведены исходная и итоговая статистика аннотаций.

---

## Заключение

Скрипт **lateralus** обеспечивает автоматизированное улучшение аннотаций генома посредством сравнения CDS‑записей из целевого и референсного файлов. Он использует современные алгоритмы глобального выравнивания для оценки идентичности белковых последовательностей, обновляет ключевые поля аннотаций, регистрирует все изменения и выполняет параллельную обработку для повышения производительности.  
Данный скрипт был создан с помощью Copilot для демонстрации возможностей автоматизации и оптимизации биоинформатических рабочих процессов.

Если у вас возникнут вопросы или потребуется адаптация функционала, документация и исходный код готовы для дальнейшего расширения.
