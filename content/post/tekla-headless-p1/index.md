---
title: Headless? Ты что, голову дома забыл?
description: Следующий уровень в автоматизации
slug: tekla-headless-p1
date: 2024-02-21 07:00:00+0000
math: false

categories:
    - tekla

tags:
    - headless

image: cover.png

draft: false
---

## BimPublisher как автоматизация

Для Tekla есть самый популярный вариант полной автоматизации - это [BimPublisher](https://warehouse.tekla.com/#/catalog/details/u1ea375b5-0819-40e4-8105-5f3d74474352). Из названия понятно, что BimPublisher предназначен для публикации чего-то. И его даже можно использовать для других задач, не относящихся к экспорту. Для этих задач мы можем использовать макросы, которые запускаются перед экспортом (в BimPublisher это *Run these macros before exports*) и после экспорта (в BimPublisher это *Run these macros after exports*). Пример такой автоматизации можно найти [в статье из раздела Tekla User Assistance](https://support.tekla.com/article/automatically-update-your-shared-model) про автоматическое обновление моделей из Model Sharing.

## Проблема BimPublisher

Почему бы тогда не остановится на BimPublisher и пользоваться им для наших задач, зачем что-то придумывать? Проблема в том, что BimPublisher запускает Tekla с UI, который нужен человеку. Но у нас-то не будет никакого человека, который будет пользоваться Tekla, Tekla будет использоваться программой - т.е. нашим скриптом. Следовательно UI нам не нужен, но запуск Tekla и открытие модели большей частью состоит из загрузки UI.

## Headless Tekla

Решением проблемы является Headless Tekla, пример можно найти в [официальном репозитории Trimble на GitHub](https://github.com/TrimbleSolutionsCorporation/HeadlessTeklaStructuresExample). Headless Tekla находится в сборке `Tekla.Structures.Service`, а ее API не выпущен официально, поэтому в справке отсутствует документация. Но благодаря примеру можно быстро разобраться что к чему. Немного изменю файл `Program.cs` из примера и приведу его здесь с комментариями:

```csharp
var binDir = $@"C:\TeklaStructures\2022.0\bin";

// Подписка на событие, которое возникает, когда не найдена dll. 
// Поскольку при запуске Tekla нужно много dll из папки /bin/, 
// мы их не копируем к нашему исполняемому файлу, 
// а просто указываем где .Net Framework искать dll, 
// которую он не нашел в папке с исполняемым файлом.
AppDomain.CurrentDomain.AssemblyResolve += 
    (_, args) => FindAssembly(args, binDir);

// Создаем экземпляр headless tekla, в этот момент происходит запуск Tekla
using var service = new TSS.TeklaStructuresService(
    new DirectoryInfo(binDir),
    "ENGLISH",
    new FileInfo(
        $@"C:\TeklaStructures\2022.0\Environments\default\env_Default_environment.ini"
    ),
    new FileInfo($@"C:\TeklaStructures\2022.0\Environments\default\role_Steel_Detailer.ini")
);

var modelPath = @"C:\TeklaStructuresModels\test-model";
service.Initialize(new DirectoryInfo(modelPath));

// После инициализации Tekla нам доступен API
Console.WriteLine($"Connected: {new TSM.Model().GetConnectionStatus()}");
Console.WriteLine($"Model name: {new TSM.Model().GetInfo().ModelName}");
Console.ReadKey();

return;

static Assembly FindAssembly(ResolveEventArgs args, string binDir)
{
    var requestedAssembly = new AssemblyName(args.Name);
    var dllPath = Path.Combine(binDir, requestedAssembly.Name + ".dll");
    return File.Exists(dllPath) ? Assembly.LoadFile(dllPath) : throw new ArgumentException();
}
```

Этот код можно использовать чтобы открыть модель Tekla в headless режиме, после чего можно обращаться к открытой модели с помощью Tekla Open API. В следующей статье рассмотрим какие есть преимущества у этого подхода с точки зрения производительности.