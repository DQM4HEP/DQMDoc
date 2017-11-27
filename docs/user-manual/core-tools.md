

# The logging library

The logging library is currently based on the super-fast logging library [spdlog](https://github.com/gabime/spdlog). It is faster than most of the c++ logging library, threadsafe and really easy to use.

DQM4hep have defined simple macros that allows you to log from any piece of code, with different log levels. You just need to include the file *dqm4hep/Logging.h* to use them. Here is a simple example demonstrating its use :

```cpp
#include <dqm4hep/Logging.h>

int main() {
  
  dqm_debug( "This is a debug message" );
  dqm_trace( "This is a trace message" );
  dqm_info( "This is an info message" );
  dqm_warning( "This is a warning message" );
  dqm_error( "This is a error message" );
  dqm_critical( "This is a critical message" );
  
  return 0;
}
```

The spdlog library uses the [fmt library](https://github.com/fmtlib/fmt) to format optional argument in the logging macros :

```cpp
#include <dqm4hep/Logging.h>
#include <typeinfo>

int main() {
  
  int nErrors = 42;
  dqm_error( "This is a error message that occured {0} times", nErrors );
  
  try {
    throw std::runtime_error( "Invalid pointer !" );
  }
  catch(std::exception &e) {
    dqm_warning( "Caught exception of type {0}: {1}", typeid(e).name(), e.what() );
  }
  
  return 0;
}
```

The DQM4hep logging component is generally configured at the begining of the each main. Different possibilities are provided by *spdlog* to log messages :

- **Simple file** : log messages in a simple file.
- **Rotating file** : log messages into rotating files. A maximum file size and number of files are defined. When a file reaches its maximum size, a new file is opened. When the maximum number of files is reached, the first file is opened and overwritten.
- **Daily file** : log messages into rotating files. A new file is opened every N hours and N minutes.
- **Console** : log message into the console.
- **Colored console** : log message into the console. The message are colored according to the log level in use.

The *dqm4hep::core::Logger* class defines simple static methods to create a logger with different configuration :

```cpp
#include <dqm4hep/Logging.h>
#include <dqm4hep/Logger.h>
#include <typeinfo>

using namespace dqm4hep::core;

int main() {
  
  // this creates a logger to log into the
  // console and into a simple file
  Logger::createLogger("my-logger", {
    Logger::coloredConsole(),
    Logger::simpleFile("logfile.log")
  });
  
  Logger::setMainLogger("my-logger");
  
  dqm_info( "This line will appear in the console and in the file logfile.log" );
  
  return 0;
}
```

In addition to the logging macros demonstrated in the first piece of code, other ones have been defined to directly log using a user specific logger :

```cpp
#include <dqm4hep/Logging.h>
#include <dqm4hep/Logger.h>

using namespace dqm4hep::core;

int main() {
  
  // this creates a logger to log into the
  // console and into a simple file
  auto logger = Logger::createLogger("my-logger", {
    Logger::coloredConsole(),
    Logger::simpleFile("logfile.log")
  });
  
  // log in the created logger
  dqm_logger_info( logger, "This line will appear in the console and in the file logfile.log" );
  
  // log into the default logger
  dqm_info( "This line will appear only in the console (main logger)" );
  
  // change logger and log again
  Logger::setMainLogger("my-logger");
  dqm_warning( "This line will appear (again) in the console and in the file logfile.log" );
  
  // switch back to the main logger
  Logger::setMainLogger("main");
  dqm_warning( "This line will appear (again) only in the console (main logger)" );
  
  return 0;
}
```

It is also possible to log with a general macro an by passing the log level by argument :


```cpp
#include <dqm4hep/Logging.h>
#include <dqm4hep/Logger.h>

using namespace dqm4hep::core;

int main() {
  
  // this creates a logger to log into the
  // console and into a simple file
  auto logger = Logger::createLogger("my-logger", {
    Logger::coloredConsole(),
    Logger::simpleFile("logfile.log")
  });
  
  // log in the created logger
  dqm_logger_log( logger, spdlog::level::info, "This is an info message in the 'my-logger' logger !" );
  
  // log into the default logger
  dqm_log( spdlog::level::error, "This is an error message in the main logger !" );
    
  return 0;
}
```

By default, in most of the binaries provided by the DQM4hep packages, the main logger is configured to be a colored console.

# Signal mechanism

The DQM4hep core library provide a class with a similar behavior as the Qt signal/slot mechanism. This is mainly used by the DQM4hep net library to disptach data received through the network to different callback functions, but can be re-used in many contexts. The main API class is *dqm4hep::core::Signal*. The following code illustrates how to use of this class with a simple sender/receive pattern:


```cpp
#include <dqm4hep/Logging.h>
#include <dqm4hep/Signal.h>

using namespace dqm4hep::core;

///< A simple sender class
///< Send user-defined or random data using signal
class Sender {

public:
  void sendRandom() {

    int randomNumber = rand();
    std::stringstream ss; ss << "A random number : " << randomNumber;
    
    // Emit signal will notify all listeners 
    m_signal.process( ss.str() );
  }
  
  void send(const std::string &data) {
    
    // Emit signal will notify all listeners 
    m_signal.process( data );
  }
  
  Signal<const std::string &>& onSend() {
    return m_signal;
  }
  
private:
  Signal<const std::string &>    m_signal;
};

///< A simple receiver class
class Receiver {

public:
  void logData(const std::string &data) {
    dqm_info( "Received data: {0}", data );
  }
};


int main() {
  
  // Our main objects
  Sender sender;
  Receiver receiver;
  
  // connect the sender signal to the logData() method of the Receiver class
  sender.onSend().connect( &receiver, &Receiver::logData );
  
  // send data will call signal.process() and notify the receiver
  sender.sendRandom();
  sender.send( "Punk is not dead !" );
    
  return 0;
}
```

Note in this example the signature of the template parameter of the *dqm4hep::core::Signal* class :

```cpp
Signal<const std::string &>  m_signal;
``` 

As c++ is a strongly typed language, the signal template parameters signature has to be the same as the one of the method to connect. In this example, the function *logData()* has a *const std::string &* signature in argument. This example would have failed if the signature would have been different.

Note also that the *dqm4hep::core::Signal* class allows to :

- connect methods from any c++ class.
- connect methods with unlimited number of arguments. Nevertheless, the parameter types have to be specified in the template arguments of the *dqm4hep::core::Signal* object : 

```cpp
  Signal<std::string, int, int, double, float, int> aLongSignal;
``` 


  
