![alt text](image-10.png)


# Kubernetes Pod là gì?
Pod là thành phần cơ bản nhất để deploy và chạy một ứng dụng, được tạo và quản lý bởi kubernetes. Pod được dùng để nhóm (group) và chạy một hoặc nhiều container lại với nhau trên cùng một worker node, những container trong một pod sẽ chia sẻ chung tài nguyên với nhau. Thông thường chỉ nên run Pod với 1 container (mình sẽ giải thích về việc khi nào nên chạy một pod một container và một pod nhiều container ở bài khác)

![alt text](image.png)

- Vậy tại sao là lại dùng Pod để chạy container, sao không chạy container trực tiếp? Kubernetes Pod như một wrapper của container, cung cấp cho chúng ta thêm nhiều chức năng để quản lý và chạy một container, giúp container của ta chạy tốt hơn là chạy container trực tiếp, như là group tài nguyên của container, check container healthy và restart, chắc chắn ứng dụng trong container đã chạy thì mới gửi request tới container đó, cung cấp một số lifecycle để ta có thể thêm hành động vào Pod khi Pod chạy hoặc shutdown, v...v... Và kubernetes sẽ quản lý Pod thay vì quản lý container trực tiếp

![alt text](image-1.png)

# Chạy ứng dụng đầu tiên bằng Pod
Bây giờ ta bắt tay vào thực hành bài đầu tiên nào 😃. Đầu tiên ta tạo một folder và tạo một file index.js, copy đoạn code sau vào:

```js
const http = require("http");

const server = http.createServer((req, res) => {
  res.end("Hello kube\n")
});

server.listen(3000, () => {
  console.log("Server listen on port 3000")
})

```

Tạo file Dockerfile và copy đoạn code sau vào:

```Dockerfile
FROM node:12-alpine
WORKDIR /app
COPY index.js .
ENTRYPOINT [ "node", "index" ]
```

Run câu lệnh build image:

```bash
docker build . -t 080196/hello-kube
```

Test thử container có chạy đúng hay không, chạy container bằng câu lệnh:    

```bash
docker run -d --name hello-kube -p 3000:3000 080196/hello-kube
```

Gửi request tới container:

```bash
curl localhost:3000
```

Bây giờ chúng ta sẽ dùng Pod để chạy container, các bạn có thể sử dụng image 080196/hello-kube của mình hoặc tạo image của riêng các bạn theo hướng dẫn ở đây

```bash
apiVersion: v1 # Descriptor conforms to version v1 of Kubernetes API
kind: Pod # Select Pod resource
metadata:
  name: hello-kube # The name of the pod
spec:
  containers:
    - image: 080196/hello-kube # Image to create the container
      name: hello-kube # The name of the container
      ports:
        - containerPort: 3000 # The port the app is listening on 
          protocol: TCP

```

Trước hết để test Pod, ta phải expose traffic của Pod để nó có thể nhận request trước, vì hiện tại Pod của chúng ta đang chạy trong local cluster và không có expose port ra ngoài

![alt text](image-2.png)

Có 2 cách để expose port của pod ra ngoài, dùng Service resource (mình sẽ nói về Service ở bài sau) hoặc dùng kubectl port-forward. Ở bài này chúng ta sẽ dùng port-forward, chạy câu lệnh sau để expose port của pod

```bash
kubectl port-forward pod/hello-kube 3000:3000
```
![alt text](image-3.png)


Test gửi request tới pod

```bash
curl localhost:3000
```
- Nếu in ra được chữ hello kube thì pod của chúng ta đã chạy đúng. Sau khi chạy xong để clear resource thì chúng ta xóa pod bằng câu lệnh

# Tổ chức pod bằng cách sử dụng labels
- Dùng label là cách để chúng ta có thể phân chia các pod khác nhau tùy thuộc vào dự án hoặc môi trường. Ví dụ công ty của chúng ta có 3 môi trường là testing, staging, production, nếu chạy pod mà không có đánh label thì chúng ta rất khó để biết pod nào thuộc môi trường nào

![alt text](image-4.png)

![alt text](image-5.png)

- Labels là một thuộc tính cặp key-value mà chúng ta gán vào resource ở phần metadata, ta có thể đặt tên key và value với tên bất kì. Ví dụ:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-kube-testing
  labels:
    enviroment: testing # label with key is enviroment and value is testing
    project: kubernetes-series
spec:
  containers:
    - image: nginx
      name: hello-kube
      ports:
        - containerPort: 3000
          protocol: TCP

---
apiVersion: v1
kind: Pod
metadata:
  name: hello-kube-staging
  labels:
    enviroment: staging # label with key is enviroment and value is staging
    project: kubernetes-series
spec:
  containers:
    - image: nginx
      name: hello-kube
      ports:
        - containerPort: 3000
          protocol: TCP

---
apiVersion: v1
kind: Pod
metadata:
  name: hello-kube-production
  labels:
    enviroment: production # label with key is enviroment and value is production
    project: kubernetes-series
spec:
  containers:
    - image: nginx
      name: hello-kube
      ports:
        - containerPort: 3000
          protocol: TCP

```

```bash
kubectl apply -f pod-lable.yaml
```

```bash
kubectl get pod --show-labels
```

Ta có thể list pod với labels như sau
![alt text](image-6.png)

Ta có thể chọn chính xác cột label hiển thị với -L options

```bash
kubectl get pod -L enviroment
```

![alt text](image-7.png)

Và ta có thể lọc pod theo label với -l options

```bash
kubectl get pod -l enviroment=production
```

![alt text](image-8.png)


Label là một cách rất hay để chúng ta có thể tổ chức pod theo chúng ta muốn và dễ dàng quản lý pod giữa các môi trường và dự án khác nhau. 

```bash
kubectl delete -f hello-kube.yaml
```

# Phân chia `tài nguyên` của kubernetes cluster bằng cách sử dụng namespace

- Tới phần này ta đã biết cách chạy pod và dùng labels để tổ chức pod, nhưng ta chưa có phân chia `tài nguyên` giữa các môi trường và dự án khác nhau. Ví dụ trong một dự án thì ta muốn `tài nguyên` của production phải nhiều hơn của testing, thì ta làm thế nào? Chúng ta sẽ dùng namespace

- Namespace là cách để ta chia tài nguyên của cluster, và nhóm tất cả những resource liên quan lại với nhau, bạn có thể hiểu namespace như là một sub-cluster. Đầu tiên chúng ta list ra toàn bộ namespace

```bash
kubectl get ns
```

- Ta sẽ thấy có vài namespace đã được tại bởi kube, trong đó có namespace tên là default, kube-system. Namespace default là namespace chúng ta đang làm việc với nó, khi ta sử dụng câu lệnh kubectl get để hiển thị resource, nó sẽ hiểu ngầm ở bên dưới là ta muốn lấy resource của namespace mặc định. Ta có thể chỉ định resource của namespace chúng ta muốn bằng cách thêm option --namespace vào

```bash
kubectl get pod --namespace kube-system
```

Bây giờ ta sẽ thử tạo một namespace và tạo pod trong namespace đó. Cách tổ chức namespace tốt là tạo theo <project_name>:<enviroment>. Ví dụ: mapp-testing, mapp-staging, mapp-production, kala-testing, kala-production. Ở đây làm nhanh thì mình sẽ không đặt namespace theo cách trên

```bash
kubectl create ns testing
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-kube-testing
  namespace: testing # namespace name
spec:
  containers:
    - image: nginx
      name: hello-kube
      ports:
        - containerPort: 3000
          protocol: TCP

```

![alt text](image-9.png)

```bash
kubectl delete pod hello-kube-testing -n testing
kubectl delete ns testing
```