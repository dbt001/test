# Makefile分析
## 前半部分-生成可运行程序
### build-base 制作编译docker(centos)
1. docker build --build-arg IDOEWODOCKERREPO=${IDOEWODOCKERREPO} --build-arg HTTP_PROXY=${HTTP_PROXY} \
	--build-arg HTTPS_PROXY=${HTTPS_PROXY} --build-arg CNFVPP_WORK_ENV_OS_NAME=${CNFVPP_WORK_ENV_OS_NAME} \
	--build-arg CNFVPP_WORK_ENV_OS_VERSION=${CNFVPP_WORK_ENV_OS_VERSION} --rm -t ${CNFVPP_BUILD_BASE_IMAGE_NAME} \
	-f ${CNFVPPROOT}/build/Dockerfile.${CNFVPP_WORK_ENV_OS_NAME}${CNFVPP_WORK_ENV_OS_VERSION}.cnf-vpp.build-base ./  
    _使用docker file创建docker image,并同时生成EVL参数供程序运行使用_  
2. push_container.sh ${IDOEWODOCKERREPO} ${CNFVPP_BUILD_BASE_IMAGE_NAME} ${CNFVPP_BUILD_BASE_IMAGE_VERSION}  
    _上传docker image到docker hub, 其中$IDOEWODOCKERREPO由用户预定义，且docke login已成功登录_  
### pkg 编译应用程序包（依赖于build-base)
1. docker run --name ${CNFVPP_BUILD_BASE_IMAGE_NAME}-${CNFVPP_BUILD_BASE_IMAGE_VERSION} --user root:root --env GO111MODULE \
	--env XDG_CACHE_HOME=/tmp/.cache -v ${IDOEWOROOT}:/opt/ido-ewo/ -dit \
	${IDOEWODOCKERREPO}${CNFVPP_BUILD_BASE_IMAGE_NAME}:${CNFVPP_BUILD_BASE_IMAGE_VERSION} /bin/sh  
    _将第1步制作的docker(centos)启动，并将当前Host源码目录挂载到Guest /opt/ido-ewo目录下_  
2. docker exec ${CNFVPP_BUILD_BASE_IMAGE_NAME}-${CNFVPP_BUILD_BASE_IMAGE_VERSION} /bin/sh -c \
	"cd /opt/ido-ewo/platform/cnf-vpp;make -C src install-dep;make -C src pkg"  
    _在guest中调用make命令，即VPP相关依赖安装和编译，生成pkg包,此步骤同时包含VPP集成组件的编译，如sweetcomb/RestAPI等_  
#### 内部make分析
	install-dep:
	sudo ${INSTALL} git wget patch
	make -C vpp $@
	make -C sweetcomb $@
	_分别进行VPP和Sweetcomb编译，如下为VPP编译，通过从FDIO官网下载指定版本，并打上功能补丁后编译_
	rm -rf vpp-$(VERSION) v$(VERSION).tar.gz
	git config --global user.name dummy
	git config --global user.email dummy@example.com
	./patch_vpp-$(VERSION).sh
	git add vpp-$(VERSION)/*
	git commit -a -m "To do the commit locally but not to push only as a workaround to make vpp build system happy"
	make -C vpp-$(VERSION) UNATTENDED=yes $@

3. cp ${CNFVPPROOT}/src/sweetcomb/${PKG}/*.${PKG} ${CNFVPPROOT}/src/vpp/vpp-20.09/build-root/*vpp*.${PKG} \
	${CNFVPPROOT}/src/sweetcomb/build-root/build/sweetcomb*.${PKG} ${CNFVPPROOT}/build/packages  
    _将编译结果拷贝至Host build/packages目录_  
### service-base 制作发布的docker应用（依赖于编译出的deb/rpm包）
1. cp -rf ${CNFVPPROOT}/config ${CNFVPPROOT}/build  
	_将运行配置拷贝至build目录中，build目录中存放了Make后的deb/rpm包_
2. docker build --build-arg IDOEWODOCKERREPO=${IDOEWODOCKERREPO} --build-arg HTTP_PROXY=${HTTP_PROXY} \
	--build-arg HTTPS_PROXY=${HTTPS_PROXY} --build-arg CNFVPP_WORK_ENV_OS_NAME=${CNFVPP_WORK_ENV_OS_NAME} \
	--build-arg CNFVPP_WORK_ENV_OS_VERSION=${CNFVPP_WORK_ENV_OS_VERSION} --rm -t ${CNFVPP_SERVICE_BASE_IMAGE_NAME} \
	-f ${CNFVPPROOT}/build/Dockerfile.${CNFVPP_WORK_ENV_OS_NAME}${CNFVPP_WORK_ENV_OS_VERSION}.cnf-vpp.service-base ${CNFVPPROOT}/build  
	_使用service docker file制作docker image,并将build目录挂载_
3. 分析docker file  
	```
	RUN mkdir -p /opt/ido-ewo
	WORKDIR /opt/ido-ewo	   #设定cnf安装目录
	COPY packages/*.rpm ./	   #将packet拷贝于/opt/ido-ewo

	RUN yum install -y epel-release		#安装依赖rpm包和vpp rpm包
	RUN yum install -y net-tools pciutils numactl-libs libselinux-utils policycoreutils \
	libffi-devel python-setuptools policycoreutils-python selinux-policy-targeted \
	openssh protobuf-c pcre-devel libnl3 libev libssh python36 python3-setuptools \
	boost-filesystem mbedtls-devel mbedtls pkcs11-helper dnsmasq iproute net-tools
	RUN rpm -ivh ./*.rpm

	COPY packages/crd-ctrlr-adpt /usr/bin		#部署crd controller包-绿色版
	COPY config/startup.conf /etc/vpp/startup.conf  #其它运行所需配置文件
	COPY config/devinfo.json /etc
	COPY ./entrypoint.sh .
	COPY ./startup-datastore.sh .

	VOLUME [ "/sys/fs/cgroup" ]
	CMD [ "/usr/sbin/init" ]
	CMD [ "./entrypoint.sh" ]

	```
4. ${CNFVPPROOT}/deployments/push_container.sh ${IDOEWODOCKERREPO} ${CNFVPP_SERVICE_BASE_IMAGE_NAME} ${CNFVPP_SERVICE_BASE_IMAGE_VERSION}  
	_上传docker image到docker hub_   

## 后半部分-运行程序
```
deploy: check-env docker-reg
	@echo "Deploying IDO-EWO's cnf-vpp microservice container"
	@helm install sdewan ./deployments/helm --set spec.sdewan_image=${IDOEWODOCKERREPO}${CNFVPP_SERVICE_BASE_IMAGE_NAME}:${CNFVPP_SERVICE_BASE_IMAGE_VERSION}
	@echo "    Done."
 
```
	_Helm 包格式如下_
```
testapi-chart
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   └── service.yaml
└── values.yaml
```
_相关的Helm 介绍,参见 https://zhaohuabing.com/2018/04/16/using-helm-to-deploy-to-kubernetes/_

_Helm install采用本地helm包./deployments/helm的方式运行sdewan（由于未采用仓库管理方式，故此条命令需要在K8S Master上执行），上述命令中,-set 同步指定运行的docker image, 其它参数由helm values.yaml替换，values.yaml定义如下_
```
devs:
  - 0000:06:00.0
  - 0000:07:00.0

wan:
  wan1: GigabitEthernet6/0/0

lan:
  lan1: GigabitEthernet7/0/0

dhcp:
  server:
    ippool_start: 172.30.10.100
    ippool_end: 172.30.10.200
    ippool_gateway: 172.30.10.1
    ippool_mask: 255.255.255.0
    ippool_leasetime: 28800
    ippool_dns: 8.8.8.8
    ippool_address: 172.30.10.4/24
    ippool_subnet: 172.30.10.0/24

spec:
  sdewan_image: cnf-vpp-service

resources:
  limits:
    hugepages: 200Mi
    memory: 500Mi
  requests:
    cpu: 150m

cpu:
  maincore: 0
  corelist_workers: 2-3,6-7

```
_其中关键所需的变量参数都已定义出来，Helm会自动将templates中的所有yaml中用到的上述变量进行规换，并在install时，将Helm包上传至K8S API Server运行，K8S运行相关yaml资源时，这些参数已被替换_
