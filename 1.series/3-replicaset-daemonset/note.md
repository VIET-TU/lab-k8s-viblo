# Bài 4 - ReplicationControllers and other controller

![alt text](image.png)

- ReplicationControllers (RC) đã được thay thế bởi ReplicaSets trong phiên bản Kubernetes 1.9. Tuy nhiên để hiểu về cách hoạt động của ReplicaSets thì ta cần tìm hiểu về 

# ReplicationControllers là gì ?
- ReplicationControllers là một resource mà sẽ tạo và quản lý pod, và chắc chắn là số lượng pod nó quản lý không thay đổi và kept running. ReplicationControllers sẽ tạo số lượng pod bằng với số ta chỉ định ở thuộc tính replicas và quản lý pod thông qua labels của pod

![alt text](image-1.png)

- Các đặc điểm chính của ReplicationControllers:
    * Desired Replica Count: số lượng Pod ta muốn duy trì, RC đảm bảo số lượng Pod hiện tại luôn bằng với số lượng ta mong muốn
    * Pod Template
    * Selector: RC sử dụng một selector để xác định các pods mà nó quản lý
    * Self-healing: RC tự động thay thế các Pod bị lỗi hoặc tắt để đảm bảo số lượng Pod mong muốn luôn được duy trì

# Tại sao ta nên dùng ReplicationControllers để chạy pod?
- Chúng ta đã biết pod nó sẽ giám sát container và tự động restart lại container khi nó fail

![alt text](image-2.png)

- Vậy trong trường hợp toàn bộ worker node của chúng ta fail thì sẽ thế nào? pod nó sẽ không thể chạy nữa, và application của chúng ta sẽ downtime với người dùng

![alt text](image-3.png)

- Nếu chúng ta chạy cluster với hơn 1 worker node, RC sẽ giúp chúng ta giải quyết vấn đề này. Vì RC sẽ chắc chắn rằng số lượng pod mà nó tạo ra không thay đổi, nên ví dụ khi ta tạo một thằng RC với số lượng replicas = 1, RC sẽ tạo 1 pod và giám sát nó, khi một thằng worker node die, nếu pod của thằng RC quản lý có nằm trong worker node đó, thì lúc này thằng RC sẽ phát hiện ra là số lượng pod của nó bằng 0, và nó sẽ tạo ra thằng pod ở một worker node khác để đạt lại được số lượng 1

![alt text](image-4.png)

- Dưới đây là hình minh họa cách hoạt động của RC

![alt text](image-5.png)

- Sử dụng ReplicationControllers để chạy pod sẽ giúp ứng dụng của chúng ta luôn luôn availability nhất có thể. Ngoài ra ta có thể tăng performance của ứng dụng bằng cách chỉ định số lượng replicas trong RC để RC tạo ra nhiều pod chạy cùng một version của ứng dụng.

- Ví dụ ta có một webservice, nếu ta chỉ deploy một pod để chạy ứng dụng, thì ta chỉ có 1 container để xử lý request của user, nhưng nếu ta dùng RC và chỉ định replicas = 3, ta sẽ có 3 pod chạy 3 container của ứng dụng, và request của user sẽ được gửi tới 1 trong 3 pod này, giúp quá trình xử lý của chúng ta tăng gấp 3 lần

![alt text](image-6.png)

# Tạo một ReplicationControllers


```bash
apiVersion: v1
kind: ReplicationController
metadata:
  name: hello-rc
spec:
  replicas: 2 # number of the pod
  selector: # The pod selector determining what pods the RC is operating on
    app: hello-kube # label value
  template: # pod template
    metadata:
      labels:
        app: hello-kube # label value
    spec:
      containers:
      - image: 080196/hello-kube # image used to run container
        name: hello-kube # name of the container
        ports:
          - containerPort: 3000 # pod of the container

```

- Cấu trúc của một file config RC sẽ gồm 3 phần chính như sau:
    + label selector: sẽ chỉ định pod nào sẽ được RC giám sát
    + replica count: số lượng pod sẽ được tạo
    + pod template: config của pod sẽ được tạo


![alt text](image-7.png)

Bây giờ ta tạo RC nào

```bash
kubectl apply -f hello-rc.yaml
```

![alt text](image-8.png)

Kiểm tra xem rc của chúng ta đã chạy thành công hay chưa

```bash
kubectl get rc
```
![alt text](image-9.png)

- Nếu số lượng ở cột READY bằng với số lượng DESIRED thì chúng ta đã chạy RC thành công. Bây giờ ta kiểm tra số lượng pod được tạo ra bởi RC có đúng với số lượng chỉ định ở replicas như lý thuyết hay không

```bash
kubectl get pod
```

![alt text](image-10.png)

- Tên của pod được tạo ra bởi RC sẽ theo kiểu <replicationcontroller name>-<random>. Giờ ta sẽ xóa thử một thằng pod xem RC có tạo lại một thằng pod khác cho chúng ta như lý thuyết không. Nhớ chỉ định đúng tên pod của bạn

```bash
kubectl delete pod hello-rc-c6l8k
```
![alt text](image-11.png)

```bash
kubectl get pod
```

![alt text](image-12.png)

- Ta sẽ thấy là có một thằng cũ đang bị xóa đi, và cũng lúc đó, sẽ có một thằng pod mới được RC tạo ra, ở đây pod mới là hello-rc-mftfd. Hoạt động của thằng RC được mình họa như hình sau

![alt text](image-13.png)

# Thay đổi template của pod

- Ta có thể thay đổi template của pod và cập nhật lại RC, nhưng nó sẽ không apply cho những thằng pod hiện tại, muốn pod của ta cập nhật template mới, bạn phải xóa hết pod để RC tạo ra pod mới, hoặc xóa RC và tạo lại

![alt text](image-14.png)

- Vậy là chúng ta đã chạy được RC, bây giờ ta xóa đi nhé, để xóa RC thì các bạn dùng câu lệnh

```bash
kubectl delete rc hello-rc
```

Khi ta xóa RC thì những thằng pod nó quản lý cũng sẽ bị xóa theo

![alt text](image-15.png)


===> ReplicationController là một resource rất hữu ích để chúng ta deploy pod

# Sử dụng ReplicaSets thay thế RC

- Đây là một resource tương tự như RC, nhưng nó là một phiên bản mới hơn của RC và sẽ được sử dụng để thay thế RC. Chúng ta sẽ dùng ReplicaSets (RS) để deploy pod thay vì dùng RC, ở bài này mình nói về RC trước để chúng ta hiểu được nguồn gốc của nó, để đi phỏng vấn có bị hỏi vẫn biết trả lời

- Bây giờ ta sẽ tạo thử một thằng RS, config nó vẫn giống RC, chỉ khác một vài phần. Tạo file tên là hello-rs.yaml, copy config sau vào:

```yaml
apiVersion: apps/v1 # change version API
kind: ReplicaSet # change resource name
metadata:
  name: hello-rs
spec:
  replicas: 2
  selector:
    matchLabels: # change here 
      app: hello-kube
  template:
    metadata:
      labels:
        app: hello-kube
    spec:
      containers:
      - image: 080196/hello-kube
        name: hello-kube
        ports:
          - containerPort: 3000
```

```bash
kubectl apply -f hello-rs.yaml
```

Kiểm tra RS của chúng ta có chạy thành công hay không

```bash
kubectl get rs
```

![alt text](image-16.png)

- Nếu có 2 pod tạo ra là chúng ta đã chạy RS thành công. Đề xóa RS ta dùng câu lệnh

```bash
kubectl delete rs hello-rs
```

# So sánh ReplicaSets và ReplicationController

- RS và RC sẽ hoạt động tương tự nhau. Nhưng RS linh hoạt hơn ở phần label selector, trong khi label selector thằng RC chỉ có thể chọn pod mà hoàn toàn giống với label nó chỉ định, thì thằng RS sẽ cho phép dùng một số expressions hoặc matching để chọn pod nó quản lý.

- RS hỗ trợ cả label selector đơn giản lẫn phức tạp, thông qua các biểu thức logic. Điều này có nghĩa là RS có thể quản lý các Pod có những label khớp với một loạt các điều kiện, không chỉ những label cụ thể.

- Ví dụ, thằng RC không thể nào match với pod mà có env=production và env=testing cùng lúc được, trong khi thằng RS có thể, bằng cách chỉ định label selector như `env=*` . Ngoài ra, ta có thể dùng `operators` với thuộc tính matchExpressions như sau:

- Ví dụ về khả năng linh hoạt của RS với matchExpressions:
    + RS có thể sử dụng biểu thức so khớp để chọn Pod dựa trên nhiều điều kiện khác nhau.

```yaml
selector:
  matchExpressions:
    - key: app
      operator: In
      values:
        - kubia

```

- Trong ví dụ này, ReplicaSet sẽ quản lý tất cả các Pod có app=kubia. Sự linh hoạt nằm ở việc bạn có thể dùng các operator như In, NotIn, Exists, hoặc DoesNotExist để định nghĩa các điều kiện.


- ReplicationController:
    + Sẽ không thể quản lý các Pod với điều kiện phức tạp, ví dụ như những Pod có env=production hoặc env=testing cùng lúc.
- ReplicaSet:
    + Có thể quản lý các Pod với điều kiện env=production hoặc env=testing bằng cách sử dụng label selector dạng biểu thức logic.

```yaml
selector:
  matchExpressions:
    - key: env
      operator: In
      values:
        - production
        - testing
```
==> Biểu thức trên sẽ giúp RS chọn các Pod có env=production hoặc env=testing.

# Sử dụng DaemonSets để chạy chính xác một pod trên một worker node

- Đây là một resource khác của kube, giống như RS, nó cũng sẽ giám xác và quản lý pod theo lables. Nhưng thằng RS thì pod có thể deploy ở bất cứ node nào, và trong một node có thể chạy mấy pod cũng được. Còn thằng DaemonSets này sẽ deploy tới mỗi thằng node một pod duy nhất, và chắc chắn có bao nhiêu node sẽ có mấy nhiêu pod, nó sẽ không có thuộc tính replicas

![alt text](image-17.png)

- Ứng dụng của thằng DaemonSets này sẽ được dùng trong việc `logging` và `monitoring`. Lúc này thì chúng ta sẽ chỉ muốn có một pod monitoring ở mỗi node. Và ta cũng có thể đánh label vào trong một thằng woker node bằng cách sử dụng câu lệnh

```bash
kubectl label nodes <your-node-name> disk=ssd
```

- Sau đó ta có thể chỉ định thêm vào config của DaemonSets ở cột nodeSelector với disk=ssd. Chỉ deploy thằng pod trên node có ổ đĩa ssd. Đây là config ví dụ

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:
        disk: ssd
      containers:
        - name: main
          image: luksa/ssd-monitor

```

![alt text](image-18.png)