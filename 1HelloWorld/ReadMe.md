## Sử dụng Docker để cài đặt RabbitMQ

```dockerfile
docker run -it --rm --name rabbitmq -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

Nếu cài đặt thành công. Truy cập vào trang `http://localhost:15672`. Sử dụng `username` là guest và `password` là guest để đăng nhập.

### Tạo Go Project, sử dụng lệnh bên dưới:

```go
go mod init helloworld
```

### Tạo 2 file, producer.go và consumer.go
```go
touch producer.go consumer.go
```

### Sử dụng Go RabbitMQ client `github.com/streadway/amqp`.
Đây là thư viện chuẩn được khuyên sử dụng trên trang chủ của RabbitMQ

```go
go get github.com/streadway/amqp
```

### 1. Xử lý Producer:
#### import thư viện vào trong file `producer.go`:
```go
package main

import (
	"github.com/streadway/amqp"
)
```

#### Chúng ta sẽ viết một hàm phục vụ việc kiểm tra lỗi mỗi khi cần kết nối tới Broker:
```go
func failOnError(err error, msg string) {
	if err != nil {
		log.Fatalf("%s: %s", msg, err)
        }
}
```
#### Thực hiện kết nối tới Broker:
```go
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()
```

#### Sau khi kết nối thành công tới Broker, chúng ta sẽ tạo ra một kênh giao tiếp với Broker:
```go
ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()
```

#### Để gửi được một message, chúng ta cần khai bái một `queue`, sau đó publish message đó vào `queue`:
```go
q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")

body := "Hello World!"
err = ch.Publish(
  "",     // exchange
  q.Name, // routing key
  false,  // mandatory
  false,  // immediate
  amqp.Publishing {
    ContentType: "text/plain",
    Body:        []byte(body),
  })
failOnError(err, "Failed to publish a message")
```
### 2. Xử lý Consumer:
#### Chúng ta cũng sẽ cần tạo kết nối tời Broker và tạo ra một `channel` tương tự như ở `producer`
```go
conn, err := amqp.Dial("amqp://guest:guest@localhost:5672/")
failOnError(err, "Failed to connect to RabbitMQ")
defer conn.Close()

ch, err := conn.Channel()
failOnError(err, "Failed to open a channel")
defer ch.Close()

q, err := ch.QueueDeclare(
  "hello", // name
  false,   // durable
  false,   // delete when unused
  false,   // exclusive
  false,   // no-wait
  nil,     // arguments
)
failOnError(err, "Failed to declare a queue")
```

#### Sau đó chúng ta yêu cầu server gửi message cho chúng ta từ `queue`:
```go
msgs, err := ch.Consume(
  q.Name, // queue
  "",     // consumer
  true,   // auto-ack
  false,  // exclusive
  false,  // no-local
  false,  // no-wait
  nil,    // args
)
failOnError(err, "Failed to register a consumer")

forever := make(chan bool)

go func() {
  for d := range msgs {
    log.Printf("Received a message: %s", d.Body)
  }
}()

log.Printf(" [*] Waiting for messages. To exit press CTRL+C")
<-forever
```

### 3. Ok, Đã xong. Bây giờ chúng ta sẽ chạy để xem kết quả:
```go
go run consumer.go
```

```go
go run producer.go
```

