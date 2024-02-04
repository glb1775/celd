<script setup>
import twin from './twin';
import { ref, onMounted, defineProps } from 'vue'
import * as THREE from 'three'

import {  CSS2DObject } from 'three/examples/jsm/renderers/CSS2DRenderer.js';

import { twinStore } from '../store/index.js';

import { storeToRefs } from 'pinia'
const store = twinStore();
const { testSizeBool } = storeToRefs(store)

const sizeTagGroup = new THREE.Group();//所有标注对象插入，方便整体控制尺寸标签隐藏或显示
twin.model.add(sizeTagGroup)

// 射线拾取选择场景模型表面任意点xyz坐标
function rayChoosePoint(event, model, camera) {
    const px = event.offsetX;
    const py = event.offsetY;
    //屏幕坐标转标准设备坐标
    const x = (px / window.innerWidth) * 2 - 1;
    const y = -(py / window.innerHeight) * 2 + 1;
    const raycaster = new THREE.Raycaster();
    //.setFromCamera()在点击位置生成raycaster的射线ray
    raycaster.setFromCamera(new THREE.Vector2(x, y), camera);
    // 射线交叉计算拾取模型
    const intersects = raycaster.intersectObject(model, true);
    let v3 = null;
    if (intersects.length > 0) {
        // 获取模型上选中的一点坐标
        v3 = intersects[0].point
    }
    return v3;
}

// 标注线不进行深度测试，渲染顺序放在最后
// 3D场景中，用有粗细的线条
// 两点绘制一条直线 用于标注尺寸
function createLine(p1, p2) {
    const material = new THREE.LineBasicMaterial({
        color: 0xffff00,
        depthTest: false,//不进行深度测试，后渲染，叠加在其它模型之上(解决两个问题)
        // 1.穿过模型，在内部看不到线条
        // 2.线条与mesh重合时候，深度冲突不清晰
    });
    const geometry = new THREE.BufferGeometry(); //创建一个几何体对象
    //类型数组创建顶点数据
    const vertices = new Float32Array([p1.x, p1.y, p1.z, p2.x, p2.y, p2.z]);
    geometry.attributes.position = new THREE.BufferAttribute(vertices, 3);
    const line = new THREE.Line(geometry, material);
    return line
}



function createMesh(p, dir, camera) {
    const L = camera.position.clone().sub(p).length()
    const h = L / 30
    const geometry = new THREE.CylinderGeometry(0, L / 200, h);
    geometry.translate(0, -h / 2, 0)
    const material = new THREE.MeshBasicMaterial({
        color: 0x00ffff, //设置材质颜色
        depthTest: false,
    });
    const mesh = new THREE.Mesh(geometry, material);
    //通过四元数表示默认圆锥需要旋转的角度，才能和标注线段的方向一致
    const quaternion = new THREE.Quaternion();
    //参数dir表示线段方向，通过两点p1、p2计算即可，通过dir来控制圆锥朝向
    quaternion.setFromUnitVectors(new THREE.Vector3(0, 1, 0), dir)
    mesh.quaternion.multiply(quaternion)
    mesh.position.copy(p);
    return mesh;
}
// 能设置粗细的标注线
function createLine2(p1, p2) {
    const material = new LineMaterial({
        color: 0xffffff,
        linewidth: 0.05,//在具有大小衰减的世界单位中，否则为像素
        worldUnits: true,//linewidth对应像素
    });
    // line.worldUnits=true
    // line.needsUpdate = true;
    const geometry = new LineGeometry();
    geometry.setPositions([p1.x, p1.y, p1.z, p2.x, p2.y, p2.z]);
    const line = new Line2(geometry, material);

    return line
}

// 计算模型上选中两点的距离
function length(p1, p2) {
    return p1.clone().sub(p2).length()
}

const testBool = ref(false);//测量状态
const background = ref("rgba(0, 0, 0, 0.3)")
const visibility = ref("visible")
// onMounted(() => {
//     const nameDomArr = document.getElementsByName('pos')
//     console.log('nameDomArr', nameDomArr);

// })

const sizeBool = () => {
    // testBool.value = !testBool.value
    store.testSizeBool = !store.testSizeBool
    if (store.testSizeBool) {
        background.value = "rgba(0, 0, 0, 0.8)"
        sizeTagGroup.visible = true
        const domArr = document.body.getElementsByClassName("sizeTag")
        for (let i = 0; i < domArr.length; i++) {
            domArr[i].style.visibility = "visible"

        }
    } else {
        background.value = "rgba(0, 0, 0, 0.3)"
        // sizeTagGroup组对象包含了所有标注线段或标签可以整体隐藏
        sizeTagGroup.visible = false
        // 如果你的标签是HTML，也可以增加CSS代码隐藏所有标注文字
        const domArr = document.body.getElementsByClassName("sizeTag")
        for (let i = 0; i < domArr.length; i++) {
            domArr[i].style.visibility = "hidden"

        }
    }
}

let clickNum = 0;//记录点击次数
let p1 = null;
let p2 = null;
let L = 0

// 通过鼠标按下抬起的时间差或者说距离差，来区分判断是鼠标拖动事件，还是鼠标拖动旋转事件
let mousedownX = 0;
let mousedownY = 0;
twin.renderer.domElement.addEventListener('mousedown', function (event) {
    mousedownX = event.offsetX;
    mousedownY = event.offsetX;
})



twin.renderer.domElement.addEventListener('mouseup', function (event) {
    const x = event.offsetX;
    const y = event.offsetX;
    if (Math.abs(x - mousedownX) < 1 && Math.abs(y - mousedownY) < 1) {
        if (store.testSizeBool) {
            clickNum += 1;
            if (clickNum == 1) {
                p1 = rayChoosePoint(event, twin.model, twin.camera)
                console.log('p1', p1);
            } else {
                clickNum = 0;
                p2 = rayChoosePoint(event, twin.model, twin.camera)
                console.log('p2', p2);
                if (p1 && p2) {
                    L = length(p1, p2).toFixed(2)
                    console.log('L', L);
                    sizeTag(p1, p2, L, twin.camera);//尺寸标注 箭头线段、尺寸数值
                }
                p1 = null;
                p2 = null;
            }
        }
    }

})


function sizeTag(p1, p2, size, camera) {
    const line = createLine(p1, p2);
    sizeTagGroup.add(line)

    const dir = p1.clone().sub(p2).normalize()
    sizeTagGroup.add(createMesh(p1, dir, camera))
    sizeTagGroup.add(createMesh(p2, dir.clone().negate(), camera))

    // CSS2或CSS3渲染标注
    const div = document.createElement('div')
    div.className = 'sizeTag';
    // div.classList.add("sizeTag");
    document.body.appendChild(div)
    div.style.fontSize = "20px"
    div.style.marginTop = "-20px"
    div.style.color = "#ffffff"
    // div.style.padding = "5px 10px"
    // div.style.background = "rgba(0,0,0,0.9)"
    div.textContent = size + 'm';
    const tag = new CSS2DObject(div);
    const center = p1.clone().add(p2).divideScalar(2)
    tag.position.copy(center);
    sizeTagGroup.add(tag);
}








</script>

<template>
    <div class="pos">
        <div id="Home" class="bu" :style="{ background: background }" @click="sizeBool()">测量</div>
    </div>
</template>

<style scoped>
.pos {
    /* background-color: aqua; */
    position: absolute;
    left: 50%;
    margin-left: -40px;
    bottom: 120px;
}

.bu {
    background: rgba(0, 0, 0, 0.3);
    width: 60px;
    height: 60px;
    line-height: 60px;
    text-align: center;
    color: #fff;
    display: inline-block;
    border-radius: 30px;
}

.bu:hover {
    cursor: pointer;
}
</style>
