<script setup>
import { fabric } from "fabric";
import { onMounted, ref, watch } from "vue";

var canvas;

const penSize = ref(10);
const penColor = ref("#000000");
watch(penSize, (next, prev) => {
  canvas.freeDrawingBrush.width = next;
});
watch(penColor, (next, prev) => {
  canvas.freeDrawingBrush.color = next;
});

const drawIsActive = ref(true);

const downloadImageSize = ref("512");
const downloadImageType = ref("jpeg");


onMounted(() => {
  canvas = new fabric.Canvas("canvas", {
    freeDrawingCursor: 'none',//自作のドットカーソルを表示するために必要
    isDrawingMode: true
  });
  canvas.setBackgroundColor(
    "#ffffff",
    canvas.renderAll.bind(canvas)
  );
  //オブジェクト選択時のハンドルを太くする
  fabric.Object.prototype.set({
    borderColor: "rgb(178,204,0)",//選択枠の色
    borderScaleFactor: 6,//選択枠の太さ
    cornerSize: 20,//コーナーハンドルのサイズ
    cornerColor: "rgba(0, 0, 0, 0.5)",//コーナーハンドルの色
    transparentCorners: false,//コーナーハンドルの不透明同
    cornerStrokeColor: "rgba(255, 0, 0, 1)",//コーナーハンドルの輪郭の色
    cornerStyle: "circle"//コーナーハンドルの形（circle or rect）
  });
  // fabric.Object.prototype.cornerSize = 20
  // fabric.Object.prototype.cornerColor ="rgba(0,0,0,0.5)";
  // fabric.Object.prototype.transparentCorners=false;
  // fabric.Object.prototype.cornerStrokeColor ="rgba(255,0,0,1)";
  // fabric.Object.prototype.cornerStyle="circle";//or rect

  // canvas.isDrawingMode = true; // お絵かきモードの有効化
  canvas.freeDrawingBrush.width = penSize.value; // 描画する線の太さ、初期値
  canvas.freeDrawingBrush.color = penColor.value; // 描画する線の色、初期値

});
//カーソル用のdivの位置をマウスに追従させる
document.addEventListener("mousemove", function (e) {
  cursor.style.transform = `translate(${e.clientX}px,${e.clientY}px)`;
  // "translate(" + e.clientX + "px, " + e.clientY + "px)";
});
//canvas内でドットカーソルを表示
const mouseOverAction = (e) => {
  document.getElementById("cursor").classList.add("showDotCursor");
  document.getElementById("cursor").classList.remove("hideDotCursor");
};
//canvas外でドットカーソルを消す
const mouseOutAction = (e) => {
  document.getElementById("cursor").classList.add("hideDotCursor");
  document.getElementById("cursor").classList.remove("showDotCursor");
};
//ダウンロードの処理
const download = (e) => {
  canvas.discardActiveObject();
  canvas.requestRenderAll();
  let canvasToDL = document.getElementById("canvas");
  let memCanvas = document.createElement("canvas");
  memCanvas.width = memCanvas.height = parseInt(downloadImageSize.value);
  let contextMemCanvas = memCanvas.getContext('2d');
  contextMemCanvas.drawImage(canvasToDL, 0, 0, memCanvas.width, memCanvas.height);

  let link = document.createElement("a");

  if (downloadImageType.value == "jpeg") {
    link.href = memCanvas.toDataURL("image/jpeg");
    link.download = "drawing.jpg";
    link.click();
  } else {
    link.href = memCanvas.toDataURL("image/png");
    link.download = "drawing.png";
    link.click();
  }
};
//全消去の処理
const clearObj = () => {
  canvas.clear();
  canvas.setBackgroundColor(
    "#ffffff",
    canvas.renderAll.bind(canvas)
  );
  undo_history.push(JSON.stringify(canvas));//UNDO処理
  undoBtn.removeAttribute("disabled");

};
//編集モードボタンの処理
const editMode = (e) => {
  if (canvas.isDrawingMode) {
    canvas.isDrawingMode = false;
    drawIsActive.value = false;
  }
  else {
    canvas.isDrawingMode = true;
    drawIsActive.value = true;
    canvas.discardActiveObject();
    canvas.requestRenderAll();
  }
};
const deleteObj = () => {
  canvas.getActiveObjects().forEach(element => {
    canvas.remove(element);
  });
  canvas.discardActiveObject();
  canvas.requestRenderAll();
};
//ファイルのロード
const loadFile = async (e) => {
  const img = new Image();
  img.onload = function () {
    //☆canvsサイズを画像に合わせる
    //canvasFav.setDimensions({ width: img.width, height: img.height })
    //canvasFav.renderAll()
    //☆画像描画
    fabric.Image.fromURL(img.src, function (oImg) {
      //☆画像をcanvasサイズに合わせる
      if (img.width > img.height) {
        oImg.scaleToWidth(canvas.width)
      } else {
        oImg.scaleToHeight(canvas.height)
      };
      oImg.left = (canvas.width / 2);
      oImg.top = (canvas.height / 2);
      oImg.originX = "center";
      oImg.originY = "center";
      //☆画像描画の倍率
      //oImg.scale(1);
      canvas.add(oImg)
      //☆枠線をなくす
      oImg.hasBorders = true;
      //☆枠線の□をなくす
      oImg.hasControls = true;
    });
  };
  img.src = URL.createObjectURL(e.target.files[0]);
};
</script>
<template>
  <div class="grid grid-flow-col my-2 gap-4 px-2">
    <div class="flex justify-center" v-bind:class="{'bg-sky-200':drawIsActive, 'bg-red-200':!drawIsActive}">
      <div v-on:mouseover="mouseOverAction" v-on:mouseout="mouseOutAction" class="relative">
        <canvas id="canvas" width="512" height="512" class="border absolute"></canvas>
      </div>
    </div>
    <div>
      <div class="flex justify-center">
        <button id="editMode" class="button" v-bind:class="{'bg-sky-200':drawIsActive, 'bg-red-200':!drawIsActive}"
          v-on:click="editMode">
          <p v-if="drawIsActive">DRAW MODE (Click to EDIT MODE)</p>
          <p v-if="!drawIsActive">EDIT MODE (Click to DRAW MODE)</p>
        </button>
      </div>
      <div class="border my-2">
        <label for="loadFile" class="">Uploading Image</label><br />
        <input type="file" id="loadFile" v-on:change="loadFile" required minlength="4" maxlength="8" size="10">
      </div>
      <div class="bg-gray-300">
        <div class="penStateInput">
          <label class="penStateInputLabel">Pen Size</label>
          <input type="number" v-model="penSize" name="PenSize" class="border">
        </div>
        <div class="penStateInput">
          <label class="penStateInputLabel">Pen Color</label>
          <input type="color" v-model="penColor" name="PenSize" class="border">{{ penColor }}
        </div>
      </div>
      <div>
        <button v-on:click="deleteObj" class="button bg-gray-200">Delete</button>
        <button v-on:click="clearObj" class="button bg-gray-200">Clear</button>
      </div>
      <div class="border border-gray-200 rounded">
        <button v-on:click="download" class="button bg-gray-200">Download</button>
        <div class="flex">
          <div class=" m-2">
            <p>Size</p>
            <input type="radio" id="512" value="512" v-model="downloadImageSize">
            <label for="one">512×512</label>
            <br>
            <input type="radio" id="256" value="256" v-model="downloadImageSize">
            <label for="two">256×256</label>
            <br>
          </div>
          <div class=" m-2">
            <p>Type</p>
            <input type="radio" id="imageJpeg" value="jpeg" v-model="downloadImageType">
            <label for="one">Jpeg</label>
            <br>
            <input type="radio" id="imagePing" value="ping" v-model="downloadImageType">
            <label for="two">Ping</label>
            <br>
          </div>
        </div>
      </div>
    </div>
  </div>
  <div id="cursor" class="hideDotCursor"></div>
</template>
<style scoped>
.penStateInput {
  @apply p-2
}

.penStateInputLabel {
  @apply mx-2
}

.button {
  @apply border rounded-xl m-2 text-lg p-2
}

#cursor {
  --cursor-color-rgb: 0, 0, 0;
  --cursor-offset: -2.5px;
  --cursor-size: 5px;
  transform: translate(0, 0);
  pointer-events: none;
  position: fixed;
  top: var(--cursor-offset);
  left: var(--cursor-offset);
  width: var(--cursor-size);
  height: var(--cursor-size);
  background: rgb(var(--cursor-color-rgb));
  border-radius: 50%;
  z-index: 999;
  border: 1px solid #606060;
}

.showDotCursor {
  opacity: 0.7;
}

.hideDotCursor {
  opacity: 0;
}
</style>