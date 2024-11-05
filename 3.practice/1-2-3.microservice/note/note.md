# Triển khai nodejs microservice trên Kubernetes - phần 1 - Viết config

- Chào mọi người đến với series về kubernetes của mình, trong series này mình sẽ chia sẻ cho các bạn những kinh nghiệm của mình khi triển khai các ứng dụng thực tế trên môi trường kubernetes. Ở phần này mình sẽ nói về cách triển khai nodejs microservice trên môi trường kubernetes, phần một sẽ nói về cách viết file cấu hình từng thành phần cần thiết của microservice, phần hai sẽ nói về cách làm sao để sử dụng gitlab-ci để triển khai ứng dụng một cách tự động.

- Về hệ thống, phần backend mình sẽ sử dụng nodejs molecular framework, đây là một framework dùng để xây dựng một ứng dụng với kiến trúc microservice.

![alt text](image-2.png)

![alt text](image.png)

# Kiến trúc của ứng dụng

![alt text](image-1.png)

- API Gateway, sẽ đóng vai trò là ngõ vào cho ứng dụng của ta, nó sẽ tạo một http endpoint và lắng nghe request từ client.

- NATS, sẽ đóng vai trò là một transporter, nó sẽ như là một trạm chuyển tiếp để cho các service có thể giao tiếp được với nhau.

- Categories service, News service, đây là service thực hiện công việc CRUD resource liên quan.

- Redis, dùng để cache kết quả lấy ra từ database và kết quả trả về cho client, giúp giảm số lần thực hiện query vào DB và tăng tốc độ của ứng dụng.

- Database thì ta sẽ xài postgres.

- Mình đã nói sơ qua về kiến trúc mà ta sẽ triển khai lên trên kubernetes, bây giờ ta sẽ bắt đầu tiến hành viết config cho từng thành phần riêng biệt, bước đầu tiên là ta sẽ build image cho backend của chúng ta.

# Build image

- Các bạn có thể sử dụng image mình đã build sẵn, tên là 080196/microservice, hoặc nếu các bạn thích tự build image cho chính mình, thì các bạn tải source code từ đây xuống https://github.com/hoalongnatsu/microservices.

Sau khi tải được source code, các bạn nhảy vào thư mục microservices/code, và thực hiện build image với tên image sẽ là <docker-hub-username>/microservice, sau đó các bạn thực hiện câu lệnh docker login và push image lên docker hub của mình.

```bash
git clone https://github.com/hoalongnatsu/microservices.git && cd microservices/code
docker build . -t viettu123/microservice
docker push viettu123/microservice
```

- Sau khi build image và push lên docker hub thành công, ta sẽ tiến hành viết file config cho ứng dụng.

# Deploy API Gateway

- Đầu tiên ta sẽ viết file config cho api gateway, để tạo api gateway, ta sẽ dùng Deployment, Deployment là gì các bạn có thể xem ở đây, tạo file tên là `api-gateway-deployment.yaml` với config như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  labels:
    component: api-gateway
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: api-gateway
  template:
    metadata:
      labels:
        component: api-gateway
    spec:
      containers:
        - name: api-gateway
          image: viettu123/microservice
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          env:
            - name: NODE_ENV
              value: testing
            - name: SERVICEDIR
              value: dist/services
            - name: SERVICES
              value: api
            - name: PORT
              value: "3000"
            - name: CACHER
              value: redis://redis:6379
```

- Trong image `viettu123/microservice` chúng ta đã build ở trên, sẽ có 3 service ở trong image đó là api, categories, news. Ta chọn service ta cần chạy bằng cách truyền giá trị của service ta muốn chạy vào biến môi trường có tên là `SERVICES`, ở file config trên ta chạy api gateway, nên ta truyền vào giá trị là `api`.

- Các bạn nhìn vào phần code ở file `code/services/api.service.ts`, ta sẽ thấy ở chỗ setting cho api gateway ở dòng 15

```js
...
settings: {
  port: process.env.PORT || 3001,
...
```

- Với biến môi trường PORT, ta sẽ chọn port mà api gateway lắng nghe, ở trên ta truyền vào giá trị là 3000. Vậy api gateway của ta sẽ chạy và lắng nghe ở port 3000. Biến môi trường CACHER là ta khai báo redis host mà các service của chúng ta sẽ sử dụng. Ta tạo deployment.


```bash
$ kubectl apply -f api-gateway-deployment.yaml
deployment.apps/api-gateway created

$ kubectl get deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
api-gateway   0/1     1            0           100s

```

- Ta đã tạo được api gateway, nhưng khi bạn get pod, bạn sẽ thấy nó không chạy thành công, mà sẽ bị restart đi restart lại.

```bash
$ kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
api-gateway-78f755f78d-6tjhp  0/1     Running   2          93s
```

- Ta logs pod để xem lý do tại sao nó không chạy được mà cứ bị restart.

```bash
$ kubectl logs api-gateway-78f755f78d-6tjhp

...
[2024-10-31T09:33:38.853Z] ERROR api-gateway-78f755f78d-6tjhp-28/CACHER: Error: getaddrinfo EAI_AGAIN redis
    at GetAddrInfoReqWrap.onlookup [as oncomplete] (dns.js:60:26) {
  errno: 'EAI_AGAIN',
  code: 'EAI_AGAIN',
  syscall: 'getaddrinfo',
  hostname: 'redis'
}
```

- Lỗi được hiển thị ở đây là do ta không thể kết nối được tới redis, vì ta chưa tạo redis nào cả, tiếp theo ta sẽ tạo redis.

# Deploy Redis

- Ta tạo một file `redis-deployment.yaml` với config như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    component: redis
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      component: redis
  template:
    metadata:
      labels:
        component: redis
    spec:
      containers:
        - name: redis
          image: redis
          ports:
            - containerPort: 6379
```

```bash
$ kubectl apply -f redis-deployment.yaml
deployment.apps/redis created

$ kubectl get deploy
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
api-gateway   0/1     1            0           16m
redis         1/1     1            1           54s
```

- Vậy là ta đã tạo được redis pod, tiếp theo, nếu muốn connect được tới redis pod này, ta cần phải tạo Service resource cho nó. Tạo file tên là `redis-service.yaml` với config như sau:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    component: redis
spec:
  selector:
    component: redis
  ports:
    - port: 6379
```

- Ta tạo service:

```bash
$ kubectl apply -f redis-service.yaml
service/redis created
```

- Ta Restart api gateway deployment lại để nó có thể kết nối được tới redis.

```bash
$ kubectl rollout restart deploy api-gateway
deployment.apps/api-gateway restarted
```

- Bây giờ khi ta get pod ra, ta vẫn thấy nó vẫn chưa chạy được thành công, ta logs nó tiếp để xem tại sao.


```bash
$ k logs api-gateway-7d79576778-trxvd

Sequelize CLI [Node: 12.13.0, CLI: 6.2.0, ORM: 6.6.2]

Loaded configuration file "migrate/config.js".
Using environment "testing".


ERROR: connect ECONNREFUSED 127.0.0.1:5432
Error: Command failed: sequelize-cli db:migrate
ERROR: connect ECONNREFUSED 127.0.0.1:5432

    at ChildProcess.exithandler (child_process.js:295:12)
    at ChildProcess.emit (events.js:210:5)
    at maybeClose (internal/child_process.js:1021:16)
    at Process.ChildProcess._handle.onexit (internal/child_process.js:283:5) {
  killed: false,
  code: 1,
  signal: null,
  cmd: 'sequelize-cli db:migrate'
}
Sequelize CLI [Node: 12.13.0, CLI: 6.2.0, ORM: 6.6.2]

Loaded configuration file "migrate/config.js".
Using environment "testing".


 ERROR: connect ECONNREFUSED 127.0.0.1:5432

```

- Kết quả in ra ta thấy được là ta đã kết nối redis thành công, lỗi tiếp theo mà pod api gateway hiển thị là lỗi khi nó migrate database, nó không thể kết nối database được, vì ta chưa tạo database nào cả, bước tiếp theo là ta sẽ tạo database.

# Deploy database

- Để deploy database, ta sẽ không dùng Deployment mà sẽ dùng StatefulSet, về lý do thì các bạn có thể xem ở đây. Ta tạo một file tên là `postgres-statefulset.yaml` với config như sau:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  labels:
    component: postgres
spec:
  selector:
    matchLabels:
      component: postgres
  serviceName: postgres
  template:
    metadata:
      labels:
        component: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:11
          ports:
            - containerPort: 5432
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgres-data
          env:
            - name: POSTGRES_DB
              value: postgres
            - name: POSTGRES_USER
              value: postgres
            - name: POSTGRES_PASSWORD
              value: postgres
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: hostpath
        resources:
          requests:
            storage: 5Gi
```

- Tạo stroageClass: storageclass-hostpath.yaml

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hostpath
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

- persistentvolume-hostpath.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
  storageClassName: hostpath
```

- persistentvolumeclaim-postgres-data.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: hostpath

```



- Lưu ý phần `storageClassName`, này tùy thuộc vào kubernetes cluster của bạn thì bạn sẽ chỉ định trường storageClassName tương ứng nhé, để xem StorageClass thì bạn gõ câu lệnh sau `kubectl get sc`. Ta tạo statefulset:


```bash
$ kubectl apply -f postgres-statefulset.yaml
statefulset.apps/postgres created
$ kubectl get pod
NAME                           READY   STATUS             RESTARTS   AGE
api-gateway-79688cf6f5-g88f2   0/1     Running            16         32m
postgres-0                     1/1     Running   0                   2m10s
redis-58c4799ccc-qhv2z         1/1     Running            0          25m
```

- Ta get pod thì ta sẽ thấy postgres pod đã được tạo thành công, tiếp theo muốn kết nối được tới DB, ta cần tạo Service cho nó, tạo file tên là `postgres-service.yaml` với config như sau:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    component: postgres
spec:
  selector:
    component: postgres
  ports:
    - port: 5432
```

- Ta tạo service:

```bash
$ kubectl apply -f postgres-service.yaml
service/postgres created
```

- Giờ ta sẽ update lại api gateway Deployment để kết nối với DB ta mới tạo, các bạn xem ở trong file code/src/db/connect.ts thì sẽ thấy các biến môi trường mà api gateway dùng để kết nối tới DB.

```js
import { Op, Sequelize } from "sequelize";

export const connect = () => new Sequelize(
		process.env.DB_NAME,
		process.env.DB_USER,
		process.env.DB_PASSWORD,
		{
			host: process.env.DB_HOST,
			port: +process.env.DB_PORT,
			operatorsAliases: {
				$like: Op.like,
				$nlike: Op.notLike,
				$eq: Op.eq,
				$ne: Op.ne,
				$in: Op.in,
				$nin: Op.notIn,
				$gt: Op.gt,
				$gte: Op.gte,
				$lt: Op.lt,
				$lte: Op.lte,
				$bet: Op.between,
				$contains: Op.contains,
			},
			dialect: "postgres",
			logging: (process.env.ENABLE_LOG_QUERY === "true"),
		}
	);
```

- Ta update lại các biến env của file `api-gateway-deployment.yaml`, và tạo lại deployment.

```yaml
 ...
  env:
    - name: NODE_ENV
      value: testing
    - name: SERVICEDIR
      value: dist/services
    - name: SERVICES
      value: api
    - name: PORT
      value: "3000"
    - name: CACHER
      value: redis://redis:6379
    - name: DB_HOST
      value: postgres
    - name: DB_PORT
      value: "5432"
    - name: DB_NAME
      value: postgres
    - name: DB_USER
      value: postgres
    - name: DB_PASSWORD
      value: postgres
```

```bash
$ kubectl apply -f api-gateway-deployment.yaml
deployment.apps/api-gateway configured

$ kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
api-gateway-859cfff796-49k69   1/1     Running   0          39s
postgres-0                     1/1     Running   0          12m
redis-95c575fc6-d8j7t          1/1     Running   0          28m
```

- Bây giờ khi ta get pod, thì ta thấy pod api gateway của ta đã chạy thành công.

# Deploy categories news service

- Tiếp theo ta sẽ deploy 2 service còn lại của micoservice, tạo file tên là `categories-news-deployment.yaml` với config như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: categories-service
  labels:
    component: categories-service
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: categories-service
  template:
    metadata:
      labels:
        component: categories-service
    spec:
      containers:
        - name: categories-service
          image: viettu123/microservice
          env:
            - name: NODE_ENV
              value: testing
            - name: SERVICEDIR
              value: dist/services
            - name: SERVICES
              value: categories
            - name: CACHER
              value: redis://redis:6379
            - name: DB_HOST
              value: postgres
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: postgres
            - name: DB_USER
              value: postgres
            - name: DB_PASSWORD
              value: postgres

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: news-service
  labels:
    component: news-service
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: news-service
  template:
    metadata:
      labels:
        component: news-service
    spec:
      containers:
        - name: news-service
          image: viettu123/microservice
          env:
            - name: NODE_ENV
              value: testing
            - name: SERVICEDIR
              value: dist/services
            - name: SERVICES
              value: news
            - name: CACHER
              value: redis://redis:6379
            - name: DB_HOST
              value: postgres
            - name: DB_PORT
              value: "5432"
            - name: DB_NAME
              value: postgres
            - name: DB_USER
              value: postgres
            - name: DB_PASSWORD
              value: postgres
```

```bash
$ kubectl apply -f categories-news-deployment.yaml
deployment.apps/categories-service created
deployment.apps/news-service created
```

- Sau khi tạo 2 service này xong, để chúng và api gateway có thể giao tiếp với nhau, ta cần tạo NATS.

# Deploy NATS

- Tạo một file tên là `nats-deployment.yaml` với config như sau:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nats
  labels:
    component: nats
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      component: nats
  template:
    metadata:
      labels:
        component: nats
    spec:
      containers:
        - name: nats
          image: nats
          ports:
            - containerPort: 4222
```

```bash
$ kubectl apply -f nats-deployment.yaml
deployment/nats created
```

- Tiếp theo ta tạo service cho NATS, tạo file `nats-service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nats
  labels:
    component: nats
spec:
  selector:
    component: nats
  ports:
    - port: 4222
```

```bash
$ kubectl apply -f nats-service.yaml
service/nats created
```

- Bước cuối cùng, ta update lại env của api gateway và categories với news service, và tạo lại chúng. Thêm giá trị này vào env để cập nhật TRANSPORTER cho các service:

```yaml
...
env:
...
    - name: TRANSPORTER
      value: nats://nats:4222
```

- Cập nhật lại các deployment.

```bash
$ kubectl apply -f api-gateway-deployment.yaml
deployment.apps/api-gateway configured

$ kubectl apply -f categories-news-deployment.yaml
deployment.apps/categories-service configured
deployment.apps/news-service configured
```

- Ta get pod và xem logs của api gateway, xem các service đã có thể giao tiếp với nhau được chưa:

```bash
$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
api-gateway-7cc4589b79-m7j9d          1/1     Running   0          47s
categories-service-7669555978-77t27   1/1     Running   0          41s
nats-cd659f44b-hqqwk                  1/1     Running   0          5m23s
news-service-5bd9fc57f-lpsch          1/1     Running   0          41s
postgres-0                            1/1     Running   0          21m
redis-95c575fc6-d8j7t                 1/1     Running   0          37m


$ kubectl logs api-gateway-7cc4589b79-m7j9d
...
[2024-10-31T10:13:48.981Z] INFO  api-gateway-7cc4589b79-m7j9d-28/REGISTRY: Node 'categories-service-7669555978-77t27-28' connected.
...
[2024-10-31T10:13:52.114Z] INFO  api-gateway-7cc4589b79-m7j9d-28/REGISTRY: Node 'news-service-5bd9fc57f-lpsch-28' connected.
```

- Bạn sẽ thấy logs là `news-service` và `categories-service` đã được kết nối với api-gateway. Vậy là ứng dụng của ta đã chạy thành công, nhưng bạn có để ý thấy những biến env mà ta khai báo thì hơi dài và lập lại ở các file deployment không? Ta có thể dùng ConfigMap để khai báo cấu hình ở chỗ và sử dụng lại cho nhiều nơi, giúp file config của ta gọn hơn.

# Khai báo config chung

- Tạo một file tên là `microservice-cm.yaml` với config như sau:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: microservice-cm
  labels:
    component: microservice-cm
data:
  NODE_ENV: testing
  SERVICEDIR: dist/services
  TRANSPORTER: nats://nats:4222
  CACHER: redis://redis:6379
  DB_NAME: postgres
  DB_HOST: postgres
  DB_USER: postgres
  DB_PASSWORD: postgres
  DB_PORT: "5432"
```

```bash
$ kubectl apply -f microservice-cm.yaml
configmap/microservice-cm created
```

- Ta update lại config của các file deployment như sau, file api`-gateway-deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  labels:
    component: api-gateway
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: api-gateway
  template:
    metadata:
      labels:
        component: api-gateway
    spec:
      containers:
        - name: api-gateway
          image: 080196/microservice
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          env:
            - name: SERVICES
              value: api
            - name: PORT
              value: "3000"
          envFrom:
            - configMapRef:
                name: microservice-cm
```

```bash
$ kubectl apply -f api-gateway-deployment.yaml
deployment.apps/api-gateway configured
```

- File categories-news-deployment.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: categories-service
  labels:
    component: categories-service
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: categories-service
  template:
    metadata:
      labels:
        component: categories-service
    spec:
      containers:
        - name: categories-service
          image: 080196/microservice
          env:
            - name: SERVICES
              value: categories
          envFrom:
            - configMapRef:
                name: microservice-cm

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: news-service
  labels:
    component: news-service
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: news-service
  template:
    metadata:
      labels:
        component: news-service
    spec:
      containers:
        - name: news-service
          image: 080196/microservice
          env:
            - name: SERVICES
              value: news
          envFrom:
            - configMapRef:
                name: microservice-cm
```

```bash
$ kubectl apply -f categories-news-deployment.yaml
deployment.apps/categories-service configured
deployment.apps/news-service configured
```

- Ta get pod lại xem mọi thứ vẫn ok không:

```bash
$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
api-gateway-86b67895fd-cphmv          1/1     Running   0          79s
categories-service-84c74cd87c-zjtd2   1/1     Running   0          53s
nats-65687968fc-2drwp                 1/1     Running   0          41m
news-service-69f45b8668-kv9dm         1/1     Running   0          52s
postgres-0                            1/1     Running   0          69m
redis-58c4799ccc-qhv2z                1/1     Running   0          93m
```


# phần 2 - Automatic update config with Argocd

- Chào mọi người đến với series practice về kubernetes của mình. Tiếp tục ở bài trước (các bạn nên đọc bài trước để hiểu về hệ thống ta đang triển khai ra sao). Ở bài trước chúng ta đã nói về cách triển khai nodejs microservice trên môi trường kubernetes, ta đã nói qua về từng thành phần và cách viết config cho từng thành phần đó và cách deploy chúng lên môi trường kubernetes.

# Kiến trúc của ứng dụng

![alt text](image-3.png)

# Tự động cập nhật ứng dụng khi application template thay đổi

- Một application template sẽ chứa toàn bộ config file của tất cả các thành phần trong ứng dụng của ta.

- Các bạn sẽ để ý rằng mỗi lần ta viết file config cho một thành phần mới và thêm nó vào trong application template, ta phải chạy lại câu lệnh kubectl để tạo ra nó. Hoặc khi ta thay đổi config của một thành phần có sẵn, ta cũng phải chạy câu lệnh kubectl để cập nhật lại config cho thành phần đó.

- Thì chúng ta sẽ nói qua cách làm sao để khi ta thêm một file config mới hoặc thay đổi file config nào đó trong application template, thì ứng dụng trên kubernetes của chúng ta sẽ tự động cập nhật lại. Có rất nhiều cách để ta làm được việc này, ta có thể sử dụng jenkins, gitlab CI để thực hiện quá trình này tự động. Nhưng khi đụng vào bạn sẽ thấy có khá nhiều việc để làm.

- Ví dụ với gitlab CI, thì ta sẽ tạo một repo cho application template và nó sẽ chứa toàn bộ file config của ứng dụng, khi ta thay đổi file config nào đó, push lên gitlab, gitlab CI sẽ trigger và thực hiện quá trình CI/CD. Với gitlab thì ta sẽ thực hiện CI/CD bằng cách viết config trong file .gitlab-ci.yml, ví dụ ta có một file `.gitlab-ci.yml` như sau:

```bash
deploy template:
  stage: deploy
  tags:
    - kala
  script:
    - rsync -avz -e 'ssh -i ~/.ssh/kala.pem' . ec2-user@$SERVER:~/kubernetes --delete --force
    - |
      sudo ssh -i ~/.ssh/kala.pem ec2-user@$SERVER << EOF
        cd ~/kubernetes
        kubectl apply -f . --recursive
        exit
      EOF
  when: manual
  only:
    - production
```

- Quá trình CI/CD của gitlab đơn giản sẽ như sau. Khi ta push code lên gitlab, nó sẽ kiểm tra xem trong repo của chúng ta có file .gitlab-ci.yml hay không, nếu có thì nó sẽ trigger quá trình CI/CD này. Đầu tiên, nó pull code của repo xuống server ta chạy CI/CD và sau đó thực hiện công việc ở trường script. Ở trên trường script của ta sẽ thực hiện hai công việc như sau:
    + Công việc thứ nhất là nó sẽ thực hiện câu lệnh rsync (rsync đơn giản chỉ là một câu lệnh giúp ta chuyển file từ máy này sang máy khác), câu lệnh ở trên nó sẽ chuyển tất cả các file cấu hình ứng dụng trong repo của chúng ta lên trên server mà đang chạy kubernetes.
    + Công việc thứ hai là nó sẽ ssh tới server kubernetes master node, di chuyển vào folder ban nãy ta chuyển config file lên, và thực hiện câu lệnh kubectl apply để cập nhật lại application của chúng ta theo template mới.


- Tuy đây cũng là một cách để chúng ta cập nhật lại ứng dụng một cách tự động. Nhưng điểm yếu của cách này là khi ta chạy câu lệnh trên, toàn bộ file config đang có sẽ bị tác động hết, cái ta mong muốn là khi ta thay đổi config của thành phần nào thì chỉ thành phần đó mới được apply. Và để làm được việc đó thì ta cần phải viết một đoạn script khá dài và cũng không đơn giản chút nào. Việc này khá là tốn thời gian.

- Thì thay vì phải tự viết script để thực hiện CI/CD, thì ta có một số công cụ hỗ trợ công việc này tự động. Một trong những công cụ phổ biến nhất là Argocd.

# Argocd

- Đây là một công cụ hỗ trợ công việc tự động cập nhật lại application của chúng ta khi ta thêm hoặc thay đổi config nào đó, nhưng thay vì cập nhật lại toàn bộ như trên thì nó chỉ cập nhật lại những thành phần nào mà có thay đổi config, và tạo thêm thành phần mới nếu ta có thêm file config cho thành phần mới.

![alt text](image-4.png)

# Cài đặt Argocd

- Để cài Argocd, các bạn làm theo các bước sau đây

## 1. Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- Sau khi chạy câu lệnh xong thì ta chờ một chút để Argo CD install tất cả các Deployment và Service resource của nó.

## 2. Install Argo CD CLI

- Đây là cách cho linux:

```bash
curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

- Các môi trường khác các bạn có thể xem ở đây https://argo-cd.readthedocs.io/en/stable/cli_installation/.

## 3. Access The Argo CD API Server

- Ta chạy câu lệnh get pod để kiểm tra toàn bộ Pod của chúng ta đã chạy thành công hay chưa.

```bash
$ kubectl get pod -n argocd
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          15m
argocd-applicationset-controller-5b899f5459-2wkwp   1/1     Running   0          15m
argocd-dex-server-8586c9977f-szx4v                  1/1     Running   0          15m
argocd-notifications-controller-75d8587699-8k2fm    1/1     Running   0          15m
argocd-redis-54fcf75b8b-sl7nj                       1/1     Running   0          15m
argocd-repo-server-85977799bd-bw6vm                 1/1     Running   0          15m
argocd-server-6bf47fccf6-4srdk                      1/1     Running   0          15m
```

- Nếu tất cả đều chạy thành công, để truy cập Argo CD thì ta có thể sử dụng NodePort Service, Ingress. Ở đây để nhanh thì mình dùng Port Forwarding.

```bash
$ kubectl port-forward svc/argocd-server -n argocd 8080:443
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

- Bây giờ thì ta mở web browser và truy cập vào địa chỉ https://localhost:8080. Vì ta chạy localhost nên khi nó báo unsafe, thì ta chỉ việc bấm proceed to localhost (unsafe) là được.

## Thiết lập SSH tunnel trên máy thật
Trên máy thật, thiết lập SSH tunnel để chuyển tiếp cổng:
```bash
ssh -L 8080:localhost:8080 root@192.168.254.111
```

- Tới đây thì bạn sẽ thấy UI như sau

![alt text](image-5.png)

- Với username của ta sẽ là `admin`, và password bạn lấy bằng cách chạy câu lệnh như sau:

```bash
$ kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
9QIP4WSseh7rBK7X
```

- Sau đó ta lấy password được in ra terminal để đăng nhập. Sau khi login sau bạn sẽ thấy được giao diện.

![alt text](image-6.png)

- Nếu bạn muốn update lại password thì ta làm như sau, trước tiên chạy câu lệnh:

```bash
$ argocd login <ARGOCD_SERVER>
```

- Với ARGOCD_SERVER là IP của máy ta + port 8080 ở trên. Sau khi login sau thì ta chạy câu lệnh:

```bash
$ argocd account update-password
```

- Ok, vậy là ta đã cài đặt xong được Argocd, bước tiếp theo là ta dùng nó để tự động tạo ứng dụng và cập nhật lại ứng dụng theo application template của chúng ta.

# Tạo một App trên Argocd

- Để kết nối Argocd tới kubernetes cluster và git repo, ta cần tạo một APP trên Argocd. Nhấn vào nút `+ New App` ở trên UI.

![alt text](image-7.png)

- Bạn sẽ thấy nó mở ra một form như sau:

![alt text](image-11.png)

![alt text](image-8.png)

![alt text](image-9.png)

![alt text](image-10.png)

- Điền vào trường Application Name với tên bạn muốn, ở đây mình sẽ điền là nodejs-microservice, trường Project bạn chọn default. Trường SYNC POLICY sẽ có hai giá trị là Manual với Automatic, với Manual là khi bạn push code lên git, bạn phải vào bấm bằng tay để ứng dụng ta cập nhật lại theo template config mới, còn Automatic thì nó sẽ tự động luôn, giá trị này bạn có thể thay đổi bất cứ lúc nào cũng được. Ở đây mình sẽ chọn giá trị là Automatic. Các thuộc tính SYNC OPTIONS tạm thời ta cứ bỏ qua. Đi tiếp tới phần form tiếp theo.

![alt text](image-12.png)

![alt text](image-13.png)

- Đây là phần quan trọng, bạn sẽ điền vào trường Repository URL đường dẫn tới git repo mà bạn chứa template config của ứng dụng. Ở đây mình sẽ điền vào đường dẫn repo của mình. Trường Path, ta sẽ điền vào tên folder mà chứa file kubernetes config của ứng dụng, nếu nằm ở root thì bạn điền vào đường dẫn là /, của mình thì mình sẽ điền vào giá trị là k8s.

- Form tiếp theo ta sẽ điền vào như sau.

![alt text](image-15.png)

![alt text](image-14.png)

- Với Cluster URL bạn điền giá trị https://kubernetes.default.svc, còn namespace bạn điền tên namespace bạn muốn ứng dụng ta deploy tới. Ở đây mình điền là default. Sau khi điền sau hết, bạn nhấn nút create, lúc này Argocd sẽ tạo một APP cho chúng ta và tiến hành deploy ứng dụng lên trên kubernetes cluster của chúng ta.

![alt text](image-17.png)

![alt text](image-16.png)

Sau khi tạo xong bạn sẽ thấy UI như hiện tại, bạn để ý thấy trường status, nếu nó hiển thị giá trị synced có nghĩa là nó đã sync ứng dụng của chúng ta lên trên môi trường kubernetes thành công. Bạn có thể kiểm tra bằng cách gõ câu lệnh get deployment để xem các thành phần trong ứng dụng nodejs microservice của chúng ta.

![alt text](image-18.png)

```bash
$ kubectl get deploy
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
argocd-applicationset-controller   1/1     1            1           53m
argocd-dex-server                  1/1     1            1           53m
argocd-notifications-controller    1/1     1            1           53m
argocd-redis                       1/1     1            1           53m
argocd-repo-server                 1/1     1            1           53m
argocd-server                      1/1     1            1           53m
```

- Nếu bạn hiển thị ra được tất cả các Deployment thì chúc mừng là ta đã deploy được ứng dụng thành công. Ngoài ra Argocd còn có UI hiển thị các thành phần của chúng ta trên kubernetes kết nối với nhau ra sao và tình trạng hiện của các resource như thế nào, rất hữu ích, bạn nhấn vào APP nodejs-microservice, bạn sẽ thấy UI như sau, rất đẹp và chi tiết.

![alt text](image-19.png)

![alt text](image-20.png)

- Bây giờ thì khi bạn thay đổi một file config nào đó, push code lên github, thì Argocd sẽ tự động cập nhật lại APP cho chúng ta, ta không cần phải viết file CI/CD để thực hiện quá trình này nữa.

# Kết nối với Private Repositories

- Nếu bạn có git riêng, thì để kết nối với Private Repositories, ta làm theo bước sau, chọn icon Settings và chọn Repositories.

![alt text](image-21.png)

- Có rất nhiều lựa chọn, tùy thuộc vào trường hợp thì bạn sẽ lựa cách phù hợp.

![alt text](image-22.png)

- Ví dụ bạn bấm vào Connect repo using HTTPS thì nó sẽ hiện ra một form như sau, bạn chỉ cần điền vào thông tin chính xác là ta có thể kết nối được tới Private Repositories

![alt text](image-23.png)   

# Xóa ứng dụng

- Để xóa ứng dụng trên Argocd rất đơn giản, ở ngoài phần quản lý APP, bạn chỉ cần nhấn vào icon xóa, là tất cả mọi thứ trên APP nó trong kubernetes của chúng ta sẽ được remove đi hết.

![alt text](image-24.png)

![alt text](image-25.png)

- Bạn chạy câu lệnh get deployment thì sẽ thấy toàn bộ resource của ta đã bị xóa đi.

```bash
$ kubectl get deploy
No resources found in default namespace.
```


# phần 3 - Xây dựng CI/CD

- Chào mọi người đến với series practice về kubernetes của mình. Tiếp tục ở bài các phần trước, các bạn nên đọc bài trước để hiểu về hệ thống ta đang triển khai ra sao, phần 1 nói về cách viết config và cách triển khai microservice lên trên Kubernetes, phần 2 nói về ta dùng Argocd để tự động cập nhật application khi template config kubernetes của ta thay đổi. Ở bài này, mình sẽ nói về cách khi developer viết code cho chức năng mới của ứng dụng xong và push nó lên trên git repo, thì ta sẽ xây dựng luồng CI/CD thế nào để cập nhật lại ứng dụng với code mới của developer trên môi trường kubernetes.

- Ở bài này mình sẽ sử dụng repository là gitlab và chạy CI/CD với gitlab CI. Để làm được bài này thì các bạn cần tạo một repo trên gitlab và đẩy code lên đó nhé, các bạn đặt tên repo là gì cũng được. Các bạn đẩy code nằm ở trong folder code của repo này https://github.com/hoalongnatsu/microservices.git lên repo gitlab của các bạn nha. Sau khi xong hết ta bắt tay vào làm.

# Gitlab Runner

- Gitlab cung cấp cho ta khá nhiều cách để chạy CI/CD, mình sẽ nói tới phần đơn giản nhất trước, là ta sử dụng Gitlab Runner. Gitlab Runner là một application mà ta sẽ cài ở trên server ta muốn nó chạy CI/CD. Để install gitlab runner, các bạn ssh tới server mà mình muốn chạy CI/CD trên đó, và cài runner như sau, mình sẽ hướng dẫn cài ở trên server linux.

## Install gitlab-runner

```bash
# Download the binary for your system
sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

# Give it permission to execute
sudo chmod +x /usr/local/bin/gitlab-runner

# Create a GitLab Runner user
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash

# Install and run as a service
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

- Sau khi install xong thì bạn có thể dùng câu lệnh này để kiểm tra:

```bash
root@k8s-master-1:/home/viettu# sudo gitlab-runner list
Runtime platform                                    arch=amd64 os=linux pid=15439 revision=c6eae8d7 version=17.5.2
Listing configured runners                          ConfigFile=/etc/gitlab-runner/config.toml
```

- Câu lệnh này là để ta list toàn bộ runner hiện tại trên server, bây giờ thì tất niên khi ta list ra thì sẽ không có runner nào 😁. Giờ thì ta sẽ đăng ký một con runner để nó chạy CI/CD cho repo của ta ở trên gitlab. Bạn mở gitlab lên, ở chỗ repo của mình chọn mục Settings -> CI/CD.

![alt text](image-26.png)

- Sau đó bạn mở mục Runners ra.

![alt text](image-27.png)

- Sau khi mở ra, bạn sẽ thấy hai trường mà ta sẽ dùng nó để register runner cho repo của ta là gitlab url và token.

![alt text](image-28.png)

```bash
sudo gitlab-runner register --url https://gitlab.com/ --registration-token GR1348941JdHD8U9GySFVrEe2yD5z
```

## Register runner

- Giờ thì ta sẽ register một con runner cho repo của ta lên trên server, các bạn làm theo các bước như sau, đầu tiên ta chạy câu lệnh register.

```bash
sudo gitlab-runner register
```

- Sau khi run thì nó sẽ bắt bạn nhập gitlab url vào, nhập giá trị url ở phía trên vào. Tiếp đó nó sẽ bắt bạn nhập token, bạn nhập token ở trên vào.

![alt text](image-29.png)

- Tiếp theo là nó bắt bạn nhập description, chỗ này thì bạn nhập gì cũng được. Tiếp theo là phần tags, phần này thì quan trọng, lúc chạy CI thì runner sẽ được chạy dựa vào tags, bạn nhập ở đây là `microservice` nhé.

![alt text](image-30.png)

- Bây giờ thì sẽ tới bước ta chọn runner của ta sẽ được thực thi ra sao. Nó sẽ chạy thẳng trên server, hay là mỗi lần chạy nó sẽ tạo một container ra và chạy job CI/CD trên container đó. Chọn shell nếu các bạn muốn runner chạy thẳng trên môi trường server, chọn docker nếu bạn muốn runner chạy trong môi trường docker. Ở đây thì mình sẽ chọn `docker`, các bạn nên chọn theo mình để tránh bị lỗi nha.

![alt text](image-31.png)

- Tiếp theo nó sẽ hỏi default Docker image thì bạn nhập vào là `docker:stable`, tới đây thì ta đã register runner cho repo của ta thành công, để kiểm tra các runner ta đã đăng ký thì ta chạy câu lệnh list ở trên, giờ thì bạn sẽ thấy được runner mà ta vừa đăng ký.

```bash
$ sudo gitlab-runner list
Runtime platform                                    arch=amd64 os=linux pid=15439 revision=c6eae8d7 version=17.5.2
Listing configured runners                          ConfigFile=/etc/gitlab-runner/config.toml  
microservice                                        Executor=docker Token=t3_bAi7Hz27MydjKarCS52F URL=https://gitlab.com/
```

- Kiểm tra trên gitlab bạn sẽ thấy nó hiển thị con runner của ta.

![alt text](image-32.png)

- Oke vậy là ta đã xong bước register runner. Và thay vì phải nhập từng bước khá lằng nhằng như trên thì bạn có thể register nhanh một con runner bằng câu lệnh sau:

```bash
sudo gitlab-runner register -n \
  --url https://gitlab.com/ \
  --registration-token REGISTRATION_TOKEN \
  --executor docker \
  --description "microservice" \
  --docker-image "docker:stable" \
  --docker-privileged
```

Xóa runner

```bash
# Cách 1
sudo gitlab-runner unregister --url https://gitlab.com/ --token glrt-abcdef123456

# Cách 2
sudo nano /etc/gitlab-runner/config.toml

# Tìm và xóa phần cấu hình của runner:
[[runners]]
  name = "microservice"
  url = "https://gitlab.com/"
  token = "your-runner-token"
  executor = "shell"
  ...

```

- Giờ thì ta chỉ cần viết file CI/CD ở trong repo của chúng ta, và mỗi lần ta đẩy code lên thì file CI/CD này sẽ được con runner thực thi. Lưu ý là vì ta chọn docker nên bắt buộc trên server chạy CI/CD của ta phải có cài docker rồi thì runner mới thực thi được. Tiếp theo ta sẽ bắt tay vào làm luồng CI.

# Continuous Integration

- Đầu tiên là ta sẽ làm bước mà integrate code mới của developer với image mà dùng để chạy ứng dụng của ta trước, sau đó ta mới làm bước deploy. Ở bước integrate này thì ta sẽ viết file CI, công việc của nó sẽ là build image tương ứng với code của branch hiện tại, sau đó sẽ push image đó lên docker registery (nơi ta chứa image).

- Gitlab có hỗ trợ cho ta image registry, nên ta sẽ sử dụng nó luôn cho tiện. Để viết CI/CD cho gitlab thì ta sẽ viết trong file .gitlab-ci.yml, ở folder mà ta chứa code của repo, tạo một file .gitlab-ci.yml với config như sau:

![alt text](image-33.png)


```yaml
image: docker:stable

services:
  - docker:dind

variables:
  DOCKER_HOST: tcp://docker:2375/
  DOCKER_TLS_CERTDIR: ""
stages:
  - build

build root image:
  stage: build
  tags:
    - microservice
  script:
    - docker build . -t $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main
```

- Trường stages là ta sẽ định nghĩa những pipe trong một job của ta, hiện tại thì ta chỉ có một pipe là build thôi. Ở trong file config trên ta sẽ có biến là CI_REGISTRY_IMAGE, đây là một trong những biến mặc định của gitlab CI, biến CI_REGISTRY_IMAGE sẽ là tên image của chúng ta. Ở trên gitlab repo, bạn bấm sang phần `Package & Registries -> Container Registry`, thì bạn sẽ thấy được tên image của chúng ta. Khi ta push image thì ta sẽ push lên chỗ Container Registry này. Trường script là trường ta sẽ chỉ định những công việc ta sẽ làm, ở file trên đơn giản là ta sẽ build image và push image với code mới nhất lên.


![alt text](image-34.png)

- Giờ ta commit và push code lên, ta sẽ thấy CI/CD của ta sẽ được thực thi.

![alt text](image-35.png)

- Bạn nhấn vào chữ running thì nó sẽ dẫn bạn vào trong.

![alt text](image-36.png)

- Sau đó bạn nhấn vào chỗ build thì nó sẽ dẫn ta qua chỗ coi log.
![alt text](image-39.png)
![alt text](image-37.png)

- Runner của ta sẽ chạy và nó tự động pull code của chúng ta xuống container đang chạy job, sau đó nó sẽ thực thi build image.

![alt text](image-38.png)

- Sau khi image build xong thì nó sẽ push lên gitlab registry, và ở đây bạn sẽ thấy nó có lỗi là `denied: access forbidden`.

![alt text](image-40.png)

- Lý do là do ta chưa login tới gitlab registry để docker có thể push image lên được. Để login vào gitlab registry, ta làm như sau.

# Login gitlab registry

- Ta chọn phần Settings -> Repository, chọn mục Deploy tokens

![alt text](image-41.png)

- Ở phần Deploy tokens, bạn nhập trường name và username là microservice (này bạn muốn nhập gì cũng được).

![alt text](image-43.png)
![alt text](image-42.png)

- Chỗ scope thì bạn tạm thời chứ chọn hết. Sau đó ta nhấn tạo.

![alt text](image-45.png)
![alt text](image-44.png)

- Sau khi nhấn tạo xong thì nó sẽ hiện ra cho bạn hai trường là username và password, nhớ lưu lại nha.

![alt text](image-46.png)

```bash
username=microservice
password=gldt-9FwdFzwg7UX6LFdN28qh
```

- Sau khi có deploy token rồi, bạn chọn mục Settings -> CI/CD, mục Variables. Tạo một env là REGISTRY_PASSWORD để lưu giá trị password của deploy token.

![alt text](image-47.png)

![alt text](image-48.png)

- Ok, sau khi tạo xong thì bạn update lại file .gitlab-ci.yml như sau:


```yaml
image: docker:stable

services:
  - docker:dind

variables:
  DOCKER_HOST: tcp://docker:2375/
  DOCKER_TLS_CERTDIR: ""

stages:
  - build

build root image:
  stage: build
  tags:
    - microservice
  before_script:
    - echo "$REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u microservice --password-stdin
  script:
    - docker build . -t $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main
```

- Ta sẽ thêm trường before_script để chạy câu lệnh login vào gitlab registry. Biến CI_REGISTRY là biến mặc định của gitlab CI, biến REGISTRY_PASSWORD là ta vừa mới tạo. Cập nhật xong thì ta commit code và push nó lên lại. Lúc này bạn sẽ thấy là ta đã build được image và push nó lên thành công.

![alt text](image-49.png)

- Nhưng bạn sẽ để ý là vì ta chạy build image trong một container, nên khi ta build nó không có lưu cache, khiến việc ta build image bị lâu hơn, để khác phục việc này thì trước khi build ta sẽ pull image đã build xuống trước, để nó có layer cache, và khi ta build ta sẽ chỉ định thêm option là `--cache-from $CI_REGISTRY_IMAGE:latest`. Cập nhật lại file .gitlab-ci.yml.

```yaml
image: docker:stable

services:
  - docker:dind

variables:
  DOCKER_HOST: tcp://docker:2375/
  DOCKER_TLS_CERTDIR: ""

stages:
  - build

build root image:
  stage: build
  tags:
    - microservice
  before_script:
    - echo "$REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u microservice --password-stdin
  script:
    - docker pull $CI_REGISTRY_IMAGE:latest || true
    - docker build . --cache-from $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main

```

- Commit code và push lên, lúc này thì image của ta sẽ được build với tốc độ nhanh hơn.

# Cập nhật lại file config của kubernetes

- Tiếp theo ta sẽ cập nhật lại file config Deployment với image mà ta mới build. Để coi tên image, bạn chọn mục copy ở phần Container Registry.

![alt text](image-51.png)

![alt text](image-50.png)

- Các file Deployment ở folder k8s của repo này https://github.com/hoalongnatsu/microservices.git, như api-gateway-deployment.yaml, categories-news-deployment.yaml bạn sửa lại thành tên image của gitlab

```yaml

# api-gateway-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  labels:
    component: api-gateway
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: api-gateway
  template:
    metadata:
      labels:
        component: api-gateway
    spec:
      containers:
        - name: api-gateway
          image: registry.gitlab.com/viet-tu/argocd-microservice
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
          readinessProbe:
            httpGet:
              path: /
              port: http
          env:
            - name: SERVICES
              value: api
            - name: PORT
              value: "3000"
          envFrom:
            - configMapRef:
                name: microservice-cm
```

```yaml
# categories-news-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: categories-service
  labels:
    component: categories-service
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: categories-service
  template:
    metadata:
      labels:
        component: categories-service
    spec:
      containers:
        - name: categories-service
          image: registry.gitlab.com/viet-tu/argocd-microservice
          env:
            - name: SERVICES
              value: categories
          envFrom:
            - configMapRef:
                name: microservice-cm

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: news-service
  labels:
    component: news-service
spec:
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      component: news-service
  template:
    metadata:
      labels:
        component: news-service
    spec:
      containers:
        - name: news-service
          image: registry.gitlab.com/viet-tu/argocd-microservice
          env:
            - name: SERVICES
              value: news
          envFrom:
            - configMapRef:
                name: microservice-cm

```

- Image của mình thì mình sửa lại là registry.gitlab.com/viet-tu/argocd-microservice. Sau đó nếu bạn làm như theo bài trước ở phần 2, thì bạn chỉ cần push config lên repo chứa template config để Argocd nó tự động cập nhật lại. Còn không thì bạn chạy câu lệnh sau ở folder k8s.

```bash
$ kubectl apply -f k8s --recursive
deployment.apps/api-gateway created
deployment.apps/categories-service created
deployment.apps/news-service created
configmap/microservice-cm created
deployment.apps/nats created
service/nats created
service/postgres created
statefulset.apps/postgres created
deployment.apps/redis created
service/redis created
```

- Sau khi ta chạy câu lệnh trên thì ta đã deploy ứng dụng của ta lên kubernetes với image là image mà ta mới build và push trên gitlab. Giờ ta sẽ get pod ra để xem thử

```bash
$ kubectl get pod
NAME                                  READY   STATUS             RESTARTS   AGE
api-gateway-84677bf776-jj8x4          0/1     ErrImagePull       0          86s
categories-service-867f848c77-l2c4m   0/1     ImagePullBackOff   0          86s
nats-65687968fc-rbtd4                 1/1     Running            0          86s
news-service-686b8557c8-ch2v5         0/1     ImagePullBackOff   0          86s
postgres-0                            1/1     Running            0          86s
redis-58c4799ccc-xn9zk                1/1     Running            0          86s
```

- Nhưng khi bạn get pod ra, bạn sẽ thấy những Deployment mà ta chỉ định dùng image của gitlab thì nó sẽ bị lỗi là ErrImagePull. Bạn describe nó để xem lý do thì sẽ thấy là do những node mà pod được deploy tới không có quyền để pull image từ gitlab của ta xuống.

```bash
$ kubectl describe pod api-gateway-84677bf776-jj8x4
...
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  ...
  Normal   Pulling    61s (x4 over 2m48s)  kubelet            Pulling image "registry.kala.ai/microservice/code"
  Warning  Failed     61s (x4 over 2m37s)  kubelet            Failed to pull image "registry.kala.ai/microservice/code": rpc error: code = Unknown desc = Error response from daemon: pull access denied for registry.kala.ai/microservice/code, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
  Warning  Failed     61s (x4 over 2m37s)  kubelet            Error: ErrImagePull
  Warning  Failed     36s (x6 over 2m37s)  kubelet            Error: ImagePullBackOff
  Normal   BackOff    25s (x7 over 2m37s)  kubelet            Back-off pulling image "registry.kala.ai/microservice/code"
```

- Để kubernetes có thể pull được image từ gitlab, ta cần tạo một Secret với loại là docker-registry rồi chỉ định vào trường imagePullSecrets khi khai báo config cho Pod. Ta tạo Secret bằng câu lệnh sau.

```bash
$ kubectl create secret docker-registry microservice-registry --docker-server=https://registry.gitlab.com --docker-username=microservice --docker-password=gldt-9FwdFzwg7UX6LFdN28qh
secret/microservice-registry created
```

- Với --docker-server là tên gitlab server của bạn, --docker-username là username của deploy token ta tạo khi nãy, và --docker-password là password của deploy token. Cập nhật lại file Deployment thêm vào trường imagePullSecrets với Secret ta vừa tạo.



```yaml

# api-gateway-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
  labels:
    component: api-gateway
spec:
    ...
    spec:
      imagePullSecrets:
        - name: microservice-registry
      containers:
        - name: api-gateway
          image: registry.gitlab.com/viet-tu/argocd-microservice
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
          ...
```

```yaml
# categories-news-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: categories-service
  labels:
    component: categories-service
spec:
    ...
    spec:
      imagePullSecrets:
        - name: microservice-registry
      containers:
        - name: categories-service
          image: registry.gitlab.com/viet-tu/argocd-microservice
          ...

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: news-service
  labels:
    component: news-service
spec:
    ...
    spec:
      imagePullSecrets:
        - name: microservice-registry
      containers:
        - name: news-service
          ...

```

```bash
$ kubectl apply -f k8s --recursive
deployment.apps/api-gateway configured
deployment.apps/categories-service configured
...
statefulset.apps/postgres configured
...

$ kubectl get pod
NAME                                  READY   STATUS    RESTARTS   AGE
api-gateway-6c9f8f68b7-mj6r9          1/1     Running   0          40s
categories-service-84db8d885b-g85md   1/1     Running   0          40s
nats-65687968fc-rbtd4                 1/1     Running   0          19m
news-service-76d559db69-vqg59         1/1     Running   0          40s
postgres-0                            1/1     Running   0          19m
redis-58c4799ccc-xn9zk                1/1     Running   0          19m
```

- Vậy là ta đã hoàn thành xong bước CI, tiếp theo ta sẽ xem cách làm CD.

# Continuous deployment

- Bước này thì ta cũng sẽ có nhiều cách, mình sẽ nói về cách dễ nhất trước là ta sẽ ssh lên kubernetes master và thực hiện câu lệnh kubectl, các cách khác mình sẽ nói ở bài sau. Cập nhật lại file .gitlab-ci.yml như sau:

```yaml
image: docker:stable

services:
  - docker:dind

variables:
  DOCKER_HOST: tcp://docker:2375/
  DOCKER_TLS_CERTDIR: ""

stages:
  - build

build root image:
  stage: build
  tags:
    - microservice
  before_script:
    - echo "$REGISTRY_PASSWORD" | docker login $CI_REGISTRY -u microservice --password-stdin
  script:
    - docker pull $CI_REGISTRY_IMAGE:latest || true
    - docker build . --cache-from $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:latest -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main

deploy stage:
  stage: deploy
  tags:
    - microservice
  before_script:
    - mkdir ~/.ssh
    - echo -e "${SERVER_KEY_PEM//_/\\n}" > ~/.ssh/key.pem
    - apt-get -y update && apt-get -y install openssh-client rsync grsync
    - ssh-keyscan -H $SERVER_IP >> ~/.ssh/known_hosts
    - chmod 400 ~/.ssh/rnd-ecommerce.pem
  script:
    - |
      sudo ssh -i ~/.ssh/key.pem $SERVER_USER@$SERVER_IP << EOF
        kubectl set image deployment/api-gateway api-gateway=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        kubectl set image deployment/news-service news-service=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        kubectl set image deployment/categories-service categories-service=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
        exit
      EOF
  only:
    - main

```


- Ta sẽ thêm một stages nữa là deploy. Để ssh được tới server thì tùy cách bạn dùng là password hoặc key pem thì sẽ khác nhau. Của mình sẽ dùng cách key pem. Bạn vào phần Settings -> CI/CD -> Variables khi nãy, khai báo thêm các biến là SERVER_KEY_PEM, SERVER_USER, SERVER_IP tương ứng với server mà đang chạy kubernetes master. Khi ta build image và push lên gitlab xong, job deploy sẽ được chạy, nó sẽ ssh tới kubernetes master và cập nhật lại Deployment với image là image là mới build xong. `Vì ở đây mình làm demo nên mình gộp 3 câu cập nhật Deployment lại trong một repo, còn thực tế các bạn nên làm mỗi repo một Deployment khác nhau nhé.`

- Commit và push code lên, lúc này bạn sẽ thấy pipe của ta sẽ có 2 job là build với deploy.

![alt text](image-52.png)

![alt text](image-53.png)

- Vào xem log phần deploy.

![alt text](image-54.png)

- Ok, vậy là luồng CD của ta đã chạy thành công 😁.

# Kết luận
- Vậy là ta đã tìm hiểu xong cách xây dựng luồn CI/CD với kubernetes, khi ta xây dựng CI/CD thì công việc deploy của ta trở nên dễ dàng hơn nhiều. Nếu các bạn có thắc mắc hoặc chưa hiểu rõ chỗ nào, các bạn có thể hỏi ở phần comment. Ở phần tiếp theo mình sẽ nói về cách integrate gitlab CI với thẳng kubernetes, không cần phải chạy gitlab runner, và cách xài RBAC để thiết lập permission cho từng job với namespace cụ thể.