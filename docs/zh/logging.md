# 日志记录

Castle Project不包含其自己的日志记录框架，因为已经有很出色的框架了。“ ILogger”和“ ILoggerFactory”是对于日志记录功能的抽象，用于将Castle 库与您决定使用的框架分离。

## 日志记录器

Castle Core 提供一下日志记录器的实现, 你也可以自己创建:

记录器      | 实现
----------- | --------------
Null        | `Castle.Core.Logging.NullLogFactory` (`Castle.Core.dll`)
Console     | `Castle.Core.Logging.ConsoleFactory` (`Castle.Core.dll`)
Diagnostics | `Castle.Core.Logging.DiagnosticsLoggerFactory` (`Castle.Core.dll`)
Trace       | `Castle.Core.Logging.TraceLoggerFactory` (`Castle.Core.dll`)
Stream      | `Castle.Core.Logging.StreamLoggerFactory` (`Castle.Core.dll`)
[log4net](http://logging.apache.org/log4net/) | `Castle.Services.Logging.Log4netIntegration.Log4netFactory` (`Castle.Services.Logging.Log4netIntegration.dll`)
[log4net](http://logging.apache.org/log4net/) extended | `Castle.Services.Logging.Log4netIntegration.ExtendedLog4netFactory` (`Castle.Services.Logging.Log4netIntegration.dll`)
[NLog](http://nlog-project.org/) | `Castle.Services.Logging.NLogIntegration.NLogFactory` (`Castle.Services.Logging.NLogIntegration.dll`)
[NLog](http://nlog-project.org/) extended | `Castle.Services.Logging.NLogIntegration.ExtendedNLogFactory` (`Castle.Services.Logging.NLogIntegration.dll`)
[Serilog](http://serilog.net/) | `Castle.Services.Logging.SerilogIntegration.SerilogFactory` (`Castle.Services.Logging.SerilogIntegration.dll`)
