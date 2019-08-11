### Railway Oriented programming
Railway Oriented programming - это такой паттерн проектирования, при котором во время выполнения нашей программы мы особым образом обрабатываем ошибки.
При Railway Oriented programming:
- решаем проблему множественных if\else
- можо добавить дополнительную логику к результату операции
- позволяет написать вызове методов по цепочке

#HSLIDE
![Flux Explained](https://zohaib.me/content/images/2015/03/Screenshot-2015-03-23-01-12-31.png)
![Flux Explained](https://zohaib.me/content/images/2015/03/Screenshot-2015-03-23-01-12-38.png)
![Flux Explained](https://zohaib.me/content/images/2015/03/Screenshot-2015-03-23-01-12-45.png)

#HSLIDE
### Не ROP стиль
Проблемы
- Сложная обработка каждого шага (много if\else)
```
class NotRailway
  attr_accessor :arg

  def initialize(arg)
    @arg = arg
  end

  def call
    validation_result = validate
    return 'validation error' unless validation_result

    calcaulation_result = calculate
    return 'calcaulation error' unless calcaulation_result

    persist_result = persist
    return 'persisting error' unless persist_result

    true
  end
  
  private

  def validate
    return true if arg.is_a?(Integer)
    false
  end

  def calculate
    @arg = 100 / arg
    true
  rescue
    false
  end

  def persist
    return false if arg.negative?
    db.save(arg)
    true
  end
end

def call_process
  @result = NotRailway.new(params[:input]).call

  if @result == true
    render(:page)
  else
    loger.call(result)
    render(:error_500)
  end
end
```

#HSLIDE

### ROP стиль с исключениями

Проблемы
- эксепшены используются для обработки бизнес логики (подменяем понятие exception)
- дополнительная сложность при добавлении кастомного исключения

```
class RailwayOnExceptions
  attr_accessor :arg

  def initialize(arg)
    @arg = arg
  end

  def call
    validate
    calculate
    persist
    true
  end

  private

  def validate
    return if arg.is_a?(Integer)

    raise 'validation error'
  end

  def calculate
    @arg = 100 / arg
  rescue
    raise 'calculation error'
  end

  def persist
    raise 'persisting error' if arg.negative?

    db.save(arg)
  end

  class MyError < StandardError
  end
end

def call_process
  @result = RailwayOnExceptions.new(params[:input]).call
  render(:page)
rescue MyError => e
  loger.call(e)
  render(:error_400)
rescue StandardError => e
  render(:error_500)
end
```

#HSLIDE

### ROP стиль с использование аккумулятора 

Проблемы
- отсутсвие единого интерфеса ответов

```
class RailwayOnAccumulator
  attr_accessor :arg

  def initialize(arg)
    @arg = arg
  end

  def call
    result = %i[
      validate
      calculate
      persist
    ].reduce('') do |error, step|
      next error unless error.empty?
      send(step)
    end

    return true if result.empty?
    result
  end

  private

  def validate
    return 'validation error' unless arg.is_a?(Integer)
    ''
  end

  def calculate
    @arg = 100 / arg
    ''
  rescue
    'calcaulation error'
  end

  def persist
    return 'persisting error' if arg.negative?

    db.save(arg)
    ''
  end
end

def call_process
  @result = RailwayOnAccumulator.new(params[:input]).call

  if @result == true
    render(:page)
  else
    loger.call(result)
    render(:error_500)
  end
end
```

#HSLIDE

### ROP стиль с использованием объекта Result

Проблемы
- страшное нововведение и упоминание монад

```
class RailwayOnResultType
  attr_accessor :arg

  def initialize(arg)
    @arg = arg
  end

  def call
    Result.success(5).
      call(&method(:validate)).
      call(&method(:calculate)).
      call(&method(:persist))
  end

  private

  def validate(arg)
    return Failure.new('validation error') unless arg.is_a?(Integer)
    Success.new(arg)
  end

  def calculate
    Success.new(100 / arg)
  rescue
    Failure.new('calculation error')
  end

  def persist
    return Failure.new('persisting error') if arg.negative?

    db.save(arg)

    Success.new(arg)
  end
end

class Result
  attr_reader :value

  def initialize(v)
    @value = v
  end

  class << self
    def success(value) Success.new(value) end
    def failure(error_message) Failure.new(error_message) end
  end
end

class Success < Result
  def call(&fn)
    fn.call(@value)
  end
end

class Failure < Result
  def call(&fn)
    log(@value)
    self
  end
end

def call_process
  @result = RailwayOnResultType.new(params[:input]).call

  if result.is_a?(Success)
    render(:page)
  else
    loger.call(result.value)
    render(:error_500)
  end
end
```

#HSLIDE

### ROP стиль с библиотеками dry

Проблемы
- библиотеку новые и не факт, что их не забросят и они не начнут тянуть ваш проект назад
- опять страшное слово монады

используем 
- Dry::Transaction
- Dry::Monads
- Dry::Monads::Do

```
class MyOperation
  include Dry::Transaction
  include Dry::Monads

  step :validate
  step :log
  step :persist

  def validate(data)
    if data.valid?
      Success(name: data.name, age: data.user_age)
    else
      Failure("something went wrong")
    end
  end

  def log(name:, **rest)
    print("User name is #{name}")

    Success(name: name, **rest)
  end

  def persist(name:, age:)
    ...
    # some business logic here
    ...
    Success(name: name, age: age)
  end
end

MyOperation.new.call(...)
# ^ can return either
# Success(name: ..., age: ...)
# or Failure("something went wrong")
```

#HSLIDE

### Полезные ссылки

- https://mkdev.me/posts/primenyaem-pattern-command-pri-pomoschi-dry-transaction
- https://medium.com/@baweaver/functional-programming-in-ruby-flow-control-565bbdcdf2a2
- https://www.morozov.is/2018/05/27/do-notation-ruby.html
- https://fsharpforfunandprofit.com/rop/?source=post_page-----565bbdcdf2a2----------------------
