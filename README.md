# vue-package
#!/bin/bash

write_file(){

    #创建Dockerfile
    cat > ${WORKSPACE}/Dockerfile <<EOF
FROM nginx
MAINTAINER supernode.com
LABEL commitid=
COPY dist/ /usr/share/nginx/html/
EOF

    #创建configjs
    cat > ${WORKSPACE}/configjs <<EOF
let PLATFROM_CONFIG = {
    api: "http://192.168.1.122:8100/api", // 接口 url
    imgUrl: "http://192.168.1.122:8004", // 图片地址
    u3dUrlAPI: "http://192.168.1.122:8100", // u3d 接口请求地址
    UrlHTML: "http://192.168.1.122:6190", // u3d 访问地址
    streamingApi: "http://192.168.1.243:8080" // VR streaming 服务
};

EOF

}
#定义变量
service_name=line_mark_test_aiplatform
swarm_service_name=linemark_v2_prod_platform_line_mark_prod-java_aiplatform 
flag=`date '+%Y%m%d%H%M'`
comitid=`git log |grep commit |awk '{print $2}' | head -n 1`

npm_run_build(){
    #npm 构建
    cp -f ${WORKSPACE}/configjs ${WORKSPACE}/public/config.js
    echo "开始构建npm:"
    npm install -g cnpm -registry=https://registry.npm.taobao.org
    cnpm i
    cnpm run build
    if [ $? = 0 ];then
        echo "npm构建完成"
    else
        echo "npm构建失败"
        exit 1
    fi     
}

build_docker_images(){
    echo "制作docker镜像："
    cd ${WORKSPACE}/
    image_name=supernode:5000/${service_name}-java:${flag}
    sed -i "s/LABEL commitid=/LABEL commitid=${comitid}/" Dockerfile

    echo "*********************************************开始制作 ${service_name} 服务docker镜像**********************************************"
    docker -H tcp://192.168.1.124:4243 build -t ${image_name} .
    if [ $? = 0 ];then
        echo "docker 镜像构建成功"
    else
        echo "docker 镜像构建失败"
        exit 1
    fi 
    echo "*********************************************推送镜像：${image_name}**********************************************"
    docker -H tcp://192.168.1.124:4243 push ${image_name}
    echo "*********************************************${service_name} 服务docker镜像制作完成**********************************************"
    # tar -zcvf dist-picbiaozhu.tar.gz dist > /dev/null
    # cp -fr dist-picbiaozhu.tar.gz /var/jenkins_home/run_jar/
}

deploy_service(){
    echo "*********************************************开始部署 ${service_name} 服务**********************************************"
    docker -H tcp://192.168.1.120:4243 service update --force --image ${image_name} ${swarm_service_name}
    echo "*********************************************部署 ${service_name} 完成**********************************************"
}

run_main(){
    write_file
    npm_run_build
    build_docker_images
    deploy_service
}

run_main
