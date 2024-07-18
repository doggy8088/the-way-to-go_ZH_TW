

## 關於本文·16.10.2小結糟糕錯誤處理的一些見解

本文僅表達譯者對錯誤處理的觀點，並且覺得原文說的並不很合理，希望不會誤導（我個人觀點）其他入門讀者。

### 關於16.10.2的第一個程式碼範例

16.10.2小結中關於錯誤處理的第一個程式碼範例是標準且通用的錯誤處理方式。
文中認為這種錯誤處理方式會使你的程式碼中充滿`if err != nil {...}`，認為這樣會令人難以分辨正常的程式邏輯與錯誤處理（難道錯誤處理不算做正常的程式邏輯麼:)）。

**書中程式碼範例一**：

```Go
... err1 := api.Func1()
if err1 != nil {
    fmt.Println("err: " + err.Error())
    return
}
err2 := api.Func2()
if err2 != nil {
...
    return
}
```

**我的觀點**：

1、錯誤處理也是正常程式邏輯的一部分，程式邏輯不就是對一個操作可能出現的結果進行判斷，
並對每一種結果做相應的後續處理麼。錯誤是我們已知的可能會出現的一種結果，我們也需要處理這種情況，它也是正常邏輯的一部分。顯然，把錯誤單獨拎出來，與正常邏輯並列來做對待，並不合理。


2、在其他語言中，我們可能會用到 try... catch...語句來對可能出現的錯誤進行處理，難道你會說try-catch語句讓你的程式碼一團糟，程式邏輯和錯誤處理混在一起很複雜，讓你閱讀程式碼困難麼。絕大多數情況下，讓你感覺難以閱讀甚至噁心（可能形容過度了）的程式碼絕不會是因為錯誤處理相關的程式碼導致的，而是當時寫這些程式碼的人邏輯不清甚至邏輯混亂造成的。


3、這個可能和每個人的習慣（自己寫程式碼的思路、風格）或者說適應（看其他人的程式碼時能很快習慣作者的程式碼風格）有關，我每次看程式碼都會先略過錯誤處理的部分，那麼剩下的就是理想情況下的程式邏輯了，如果對某一處心存疑惑那麼就再仔細看這部分的程式碼。畢竟我們寫的程式碼絕大多數情況下是希望它按理想的情況跑的，

_ _ _

### 關於16.10.2的第二個程式碼範例

16.10.2小結中關於錯誤處理的第二個程式碼範例是推薦給我們的錯誤處理方式，對於其推薦的這種方式，個人認為是有一定的適用範圍的，並不適合大多數的錯誤處理，反而在處理某些業務邏輯時可以使用，比如將不符合業務邏輯的情況視作一種錯誤（自定義）來統一做處理。

**書中程式碼範例二**：

```Go
func httpRequestHandler(w http.ResponseWriter, req *http.Request) {
    err := func () error {
        if req.Method != "GET" {
            return errors.New("expected GET")
        }
        if input := parseInput(req); input != "command" {
            return errors.New("malformed command")
        }
        // 可以在此進行其他的錯誤檢測
    } ()

        if err != nil {
            w.WriteHeader(400)
            io.WriteString(w, err)
            return
        }
        doSomething() ...
```

1、程式碼範例二中對不符合業務邏輯的兩種情況做了歸類，並自定義了錯誤，做了統一的處理。這樣從業務層面來看，將不符合業務邏輯的情況視為錯誤，統一寫到了匿名函式中，剩下了一個統一的錯誤處理與正常的業務邏輯。或許採用這種方式處理這類場景還不錯，但是如果換作下面的這個範例可能就不是很合理了。

下面的範例一是採用了作者推薦的統一處理錯誤方式，範例二使用的是通常的錯誤處理方式

**範例一**：

```Go
// 目標目錄下包含多種Archive格式檔案，將其中的'x-msdownload'型別檔案移動到其他目錄下
func moveEXE(files []os.FileInfo, aimPath, exePath string) {
	var numExe, numOther int
	var fileBuf []byte
	var fileType types.Type

	for _, file := range files {
		fileName := aimPath + file.Name()
		newFileName := exePath + file.Name()

		err := func() error {  
            
            // 讀取檔案內容
			if buf, err := ioutil.ReadFile(fileName); err != nil {
				log.Printf("Time of read file: %s occur error: %s\n", fileName, err)
				return err
			}else {
				fileBuf = buf
			}
            
            // 判斷檔案是否為Archive（壓縮）格式
			if kind, err := filetype.Archive(fileBuf); err!= nil {
				log.Printf("Time of judge file type occur error: %s\n", err)
				return err
			}else {
				fileType = kind
			}
            
            // 檔案是否為'x-msdownload'型別
			if fileSubType := fileType.MIME.Subtype; fileSubType == "x-msdownload" {
				log.Printf("file : %s is exe file\n", fileName)
				if err := os.Rename(fileName, newFileName); err != nil {
					log.Printf("mv file: %s faile, error is: %s\n", fileName, err)
					return err
				}
				numExe ++
			}else {
				log.Println("no exe")
				numOther ++
			}
			return nil
		}()

		if err != nil {
			continue
		}
    }
    log.Printf("exe file num is: %d, other file num is: %d", numExe, numOther)
}
```

1、通常來說，我們使用匿名函式是因為部分操作不值得新定義一個函式或者該函式僅使用一次，範例一中的匿名函式包含了很多操作，或許我們應該為此重新定義一個函式。其中包含了幾乎全部的邏輯程式碼，我想這看起來並不是啥好主意，甚至如果你把更多的邏輯程式碼放到了匿名函式裡，看起來應該會更加糟糕。

**範例二**：

```Go

// 目標目錄下包含多種Archive格式檔案，將其中的'x-msdownload'型別檔案移動到其他目錄下
func moveEXE(files []os.FileInfo, aimPath, exePath string) {
	var numExe, numOther int

	for _, file := range files {
		fileName := aimPath + file.Name()
		newFileName := exePath + file.Name()
		
        // 讀取檔案內容
		buf, err := ioutil.ReadFile(fileName)
		if err != nil {
			log.Printf("read file:%s  occur error\n", fileName)
			continue
		}
		
        // 判斷檔案是否為Archive（壓縮）格式
		kind, err := filetype.Archive(buf)
		if err != nil {
			log.Println("judge file type error")
			continue
		}

        // 獲取檔案具體的型別
		fileSubType := kind.MIME.Subtype

        // 檔案是否為'x-msdownload'型別
		if fileSubType == "x-msdownload" {
			log.Printf("file : %s is exe file\n", fileName)
			err := os.Rename(fileName, newFileName)
			if err != nil {
				log.Printf("mv file: %s faile\n", fileName)
				continue
			}
			numExe ++
		}else {
			log.Println("no exe")
			numOther ++
		}
	}
	log.Printf("exe file num is: %d, other file num is: %d", numExe, numOther)
}
```

2、範例二中的程式碼看起來則自然多了（我是這種感覺），或許你認為這倆個例子相差無幾，但是我想透過他們表明，原文16.10.2中推薦的錯誤處理方式是有一定的使用場景的，並不能取代標準且通用的錯誤處理方式，希望大家能夠注意。

---



### 關於錯誤處理的一些延伸

1、除了使用Go中已經定義好的error，我們也可以根據需要自定義error。

下面的範例三，我們自定義了parseError 錯誤，展示了發生錯誤的檔案和具體的錯誤訊息，在你讀取目錄下的多個檔案時可以方便的告訴你具體在讀哪個檔案時發生了錯誤（作為範例，僅讀取單個檔案）。

範例四中，展示了呼叫 parseFile 函式時，呼叫者可以採用的一種錯誤處理方式，根據錯誤的型別，採取對應的操作。

**範例三**：

```go

type parseError struct {
	File *os.File
	ErrorInfo string
}


func (e *parseError) Error() string {
	errInfo := fmt.Sprintf(
		"parse file: %s occur error, error info: %s",
		e.File.Name(),
		e.ErrorInfo)
	return errInfo
}


func parseFile(path string) error {
	f, err := os.Open(path)
	if err != nil {
		return err
	}
	defer f.Close()

	var buf [512]byte
	for {
		switch num, err := f.Read(buf[:]); {
		case num < 0:
			readError := parseError{f, err.Error()}
             log.Println(readError.Error())
			return &readError
            
		case num == 0:
			readError := parseError{f, err.Error()}
             log.Println(readError.Error())
			return &readError

		case num > 0:
			fmt.Println(string(buf[:num]))
             log.Printf("read file: %s contents normally")
		}
	}
}

```

**範例四**：

```go
func main()  {
	err := parseFile("/home/rabbit/go/test_use/test")
	switch err := err.(type) {
        
	case *parseError:
        log.Println("parse error: ", err)
        
	case *os.PathError:
        log.Println("path error: ", err)
	}
}
```

2、如果你想在回傳錯誤之前做一些額外的操作，比如記錄日誌，那你可以單獨寫一個額外處理錯誤的函式或者一個匿名函式就可以（這取決於你是否常用該函式或它的功能是否很多），類似Python中的裝飾器一樣。

範例五中，handleError 將錯誤寫入到了指定日誌檔案中；

範例六中，parseFile 中使用 `defer func() {handleError("/home/rabbit/go/test_use/log", err)}()`代替了多次出現的`log.Println(readError.Error())`，並將日誌記錄持久化到檔案中。

**範例五**:

```go
func handleError(logPath string, err error) {
    if err == nil {
        return
    }
    
    logFile, _ := os.OpenFile(filepath, os.O_RDWR|os.O_APPEND|os.O_CREATE, 666)
	defer logFile.Close()

	log.SetOutput(logFile)
	log.SetPrefix("[FileError]")
	log.SetFlags(log.Llongfile|log.Ldate|log.Ltime)
	log.Println(err.Error())
}
```

**範例六**:

```go
func parseFile(path string) (err error) {
	f, err := os.Open(path)
	if err != nil {
		return err
	}
	defer f.Close()
	defer func() {handleError("/home/rabbit/go/test_use/log", err)}()

	var buf [512]byte
	for {
		switch num, err := f.Read(buf[:]); {

		case num < 0:
			err := &parseError{f, err.Error()}
			return err

		case num == 0:
			err := &parseError{f, err.Error()}
			return err

		case num > 0:
			fmt.Println(string(buf[:num]))
		}
	}
}
```

