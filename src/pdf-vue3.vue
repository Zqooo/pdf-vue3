<script lang="ts" setup>
import { ref, onUnmounted, computed, watch, onBeforeMount } from "vue";
import type { Ref } from "vue";
import type { PDFDocumentProxy } from "./index";
import * as pdfjs from "pdfjs-dist/legacy/build/pdf.min.js";
import * as workerjs from "pdfjs-dist/legacy/build/pdf.worker.min.js";
import * as pdfViewer from "pdfjs-dist/legacy/web/pdf_viewer.js";

defineOptions({
  name: "vue3-pdf-vite",
});

const dpr = ref(1);
const props = withDefaults(
  defineProps<{
    /**
     * pdf url | Uint8Array | BASE64
     */
    src: string | Uint8Array;
    httpHeaders?: Record<string, any>;
    withCredentials?: boolean;
    password?: string;
    useSystemFonts?: boolean;
    stopAtErrors?: boolean;
    disableFontFace?: boolean;
    disableRange?: boolean;
    disableStream?: boolean;
    disableAutoFetch?: boolean;
    // --custom--
    showProgress?: boolean;
    progressColor?: string;
    showPageTooltip?: boolean;
    showBackToTopBtn?: boolean;
    scrollThreshold?: number;
    pdfWidth?: string;
    rowGap?: number;
    page?: number;
  }>(),
  {
    src: undefined,
    httpHeaders: undefined,
    withCredentials: undefined,
    password: undefined,
    useSystemFonts: undefined,
    stopAtErrors: undefined,
    disableFontFace: undefined,
    disableRange: undefined,
    disableStream: undefined,
    disableAutoFetch: undefined,
    showProgress: true,
    progressColor: "#87ceeb",
    showPageTooltip: true,
    showBackToTopBtn: true,
    scrollThreshold: 300,
    pdfWidth: "100%",
    rowGap: 8,
    page: 1,
  }
);

const rowGap = computed(() => parseInt(String(props.rowGap)));

const emit = defineEmits<{
  (e: "onProgress", loadRatio: number): void;
  (e: "onComplete"): void;
  (e: "onScroll", scrollOffset: number): void;
  (e: "onPageChange", page: number): void;
  /**
   * https://mozilla.github.io/pdf.js/api/draft/module-pdfjsLib-PDFDocumentProxy.html
   */
  (e: "onPdfInit", pdf: PDFDocumentProxy): void;
}>();

const slots = defineSlots<{
  progress?: (props: { loadRatio: number }) => any;
  pageTooltip?: (props: { currentPage: number; totalPages: number }) => any;
  backToTopBtn?: (props: { scrollOffset: number }) => any;
}>();

const canvasRefs = ref<HTMLCanvasElement[]>([]);
const annotationLayerRefs = ref<HTMLElement[]>([]);

interface Option extends Record<string, any> {
  url?: string;
  data?: Uint8Array;
  httpHeaders?: Record<string, any>;
  withCredentials?: boolean;
  password?: string;
  useSystemFonts?: boolean;
  stopAtErrors?: boolean;
  disableFontFace?: boolean;
  disableRange?: boolean;
  disableStream?: boolean;
  disableAutoFetch?: boolean;
}

const loadRatio = ref(0);
const loadingTask = ref<any>(null);
let pdfContext: any = null;

// pdf文档获取, 进行配置项初始化,通过pdfjs.getDocument 获取文档内容
const getDoc = async () => {
  // 根据pdfjsLib的api进行options初始化
  const option: Option = {
    httpHeaders: props.httpHeaders,
    withCredentials: props.withCredentials,
    password: props.password,
    useSystemFonts: props.useSystemFonts,
    stopAtErrors: props.stopAtErrors,
    disableFontFace: props.disableFontFace,
    disableRange: props.disableRange,
    disableStream: props.disableStream,
    disableAutoFetch: props.disableAutoFetch,
    cMapUrl: "https://unpkg.com/pdfjs-dist@3.7.107/cmaps/",
  };
  // 判断是否为Uint8Array数组,Uint8Array -- 数组元素为一个8位服务号的整型
  if (props.src instanceof Uint8Array) {
    option.data = props.src;
    // 判断是否为pdf文件(无论是本地或接口获取,都是读取文件内容)
  } else if (props.src.endsWith(".pdf")) {
    option.url = props.src;
    // base64解析流程
  } else {
    // 通过atob进行base64解析
    const binaryData = atob(
      props.src.includes(",") ? props.src.split(",")[1] : props.src
    );
    // 根据解析内容的长度创建Uint8Array数组
    const byteArray = new Uint8Array(binaryData.length);
    // 将数据的每一位数组转换成对应的UTF-16码元,存储到容器中
    for (let i = 0; i < binaryData.length; i++) {
      byteArray[i] = binaryData.charCodeAt(i);
    }
    option.data = byteArray;
  }

  // 若options属性未定义,则删除
  for (const key in option) {
    if (option[key] === undefined) {
      delete option[key];
    }
  }

  //  渲染进度记录
  loadRatio.value = 0;
  // 进行文档获取
  loadingTask.value = pdfjs.getDocument(option);
  // 文档读取时的回调函数
  loadingTask.value.onProgress = (progressData: any) => {
    const ratio = (progressData.loaded / progressData.total) * 100;
    loadRatio.value = ratio >= 100 ? 100 : ratio;
    emit("onProgress", loadRatio.value);
  };
  // 文档读取后的回调函数
  pdfContext = await loadingTask.value.promise;
  emit("onComplete");
};

// 页面总数
const totalPages = ref(0);
// 当前页
const currentPage = ref(1);
//
const scrollOffset = ref(0);
const itemHeightList = ref<Array<number>>([]);

const scroller = ref<HTMLDivElement>() as Ref<HTMLDivElement>;
const container = ref<HTMLDivElement>() as Ref<HTMLDivElement>;

let pdfDoc: PDFDocumentProxy;
const cancelRendering = ref(false);
const renderComplete = ref(false);

//
const renderPDF = async () => {
  renderComplete.value = false;
  try {
    if (!pdfDoc) {
      // 读取pdf的文档内容,
      pdfDoc = pdfContext;

      // canvas容器绑定
      canvasRefs.value = new Array(pdfDoc.numPages);
      annotationLayerRefs.value = new Array(pdfDoc.numPages);
      totalPages.value = pdfDoc.numPages;

      // 这里不应该是pdf的初始化完成钩子，只是容器初始化
      emit("onPdfInit", pdfDoc);
    }
  } catch (error) {
    console.error("Error loadingTask PDF:", error);
  }
  let calcH = 0;
  // 根据容器总数去进行渲染逻辑，一次循环是一页pdf的渲染
  for (let i = 0; i < totalPages.value; i++) {
    try {
      // 获取当前页的代理对象，也可以理解为当前页的解析内容
      const page = await pdfDoc.getPage(i + 1);

      // ----
      //  防抖处理，若阀门开启，不继续渲染，重启当前函数执行
      if (cancelRendering.value) {
        cancelRendering.value = false;
        renderPDF();
        break;
      }

      // canvas渲染
      const canvas = canvasRefs.value[i];
      // 获取默认比例下的视口参数（595，643.5），pdf的默认是按这个视口进行canvas转换，如果容器大于渲染的pdf页面，则会放大失真
      let viewport = page.getViewport({ scale: 1 });
      // 获取当前容器与默认值的对比比例
      let scale =
        (canvas.parentNode as HTMLDivElement).clientWidth / viewport.width;
      // canvas实例获取
      const context = canvas.getContext("2d");
      // 获取当前容器的视口数据
      const scaledViewport = page.getViewport({ scale: scale * dpr.value });

      // 铆合匹配的canvas大小
      canvas.width = scaledViewport.width;
      canvas.height = scaledViewport.height;

      // 滚动相关的数据记录
      itemHeightList.value[i] = calcH +=
        scaledViewport.height / dpr.value + rowGap.value;
      // pdf渲染
      await page.render({
        canvasContext: context as CanvasRenderingContext2D,
        viewport: scaledViewport,
      });

      // 注释解析
      let viewer = {
        scrollPageIntoView: function (params: any) {
          // emitEvent("link-clicked", params.pageNumber);
          console.log(params.pageNumber);
        },
      };

      // @ts-ignore
      let linkService = new pdfViewer.PDFLinkService();
      linkService.setDocument(pdfDoc);
      linkService.setViewer(viewer);

      page
        .getAnnotations({ intent: "display" })
        .then(function (annotations: any[]) {
          const annotationLayerIns = annotationLayerRefs.value[i];
          const container = annotationLayerIns.parentElement!;
          const rect = container.getBoundingClientRect();

          annotationLayerIns.style.width = rect.width + "px";
          annotationLayerIns.style.height = rect.height + "px";
          annotationLayerIns.style.top = "0";
          annotationLayerIns.style.left = "0";

          pdfjs.AnnotationLayer.render({
            viewport,
            div: annotationLayerIns,
            annotations,
            page,
            linkService,
          });
        });
    } catch (error) {
      console.error("Error rendering PDF:", error);
    }
    if (
      props.page &&
      (i === props.page - 1 ||
        (props.page > totalPages.value && i === totalPages.value - 1))
    ) {
      scroller.value.scrollTo(0, (itemHeightList.value[i - 1] ?? 0) + 2);
    }
    if (i === totalPages.value - 1) {
      renderComplete.value = true;
    }
  }
};

// 滚动相关逻辑
const viewportHeight = ref(0);
const isScrolling = ref(false);

let scrollTimer: number;
const handleScroll = (event: any) => {
  isScrolling.value = true;
  clearTimeout(scrollTimer);
  scrollTimer = setTimeout(() => {
    isScrolling.value = false;
  }, 1000);
  scrollOffset.value = event.target.scrollTop;
  emit("onScroll", event.target.scrollTop);
  if (
    scroller.value.scrollTop + scroller.value.offsetHeight >=
    scroller.value.scrollHeight - 10
  ) {
    currentPage.value = itemHeightList.value.length;
    return;
  }

  for (let i = 0; i < itemHeightList.value.length; i++) {
    const height = itemHeightList.value[i];
    if (height > event.target.scrollTop) {
      currentPage.value = i + 1;
      break;
    }
  }
};

let timer: number;

// 渲染pdf的防抖处理
const renderPDFWithDebounce = () => {
  viewportHeight.value = window.innerHeight;
  // 判断window窗口宽度与容器宽度是否更改
  if (
    Math.abs(innerWidth.value - window.innerWidth) > 1 &&
    Math.abs(containerWidth.value - container.value.offsetWidth) > 1
  ) {
    // 如果更改，则记录当前值
    setWidth();
  } else {
    // 如果没更改，仍然记录当前值，但不重新渲染pdf
    setWidth();
    return;
  }
  // 若是频繁更改，给0.5秒缓冲时间
  cancelRendering.value = true;
  clearTimeout(timer);
  timer = setTimeout(() => {
    renderComplete.value && renderPDF();
  }, 500);
};

// 记录当前window窗口的宽度
const innerWidth = ref<number>(0);
// 记录canvas容器的宽度
const containerWidth = ref<number>(0);
// 初始化两者宽度
const setWidth = () => {
  innerWidth.value = window.innerWidth;
  if (container.value) containerWidth.value = container.value.offsetWidth;
};
const isAddEvent = ref(false);

onBeforeMount(async () => {
  // @ts-ignore
  pdfjs.GlobalWorkerOptions.workerSrc = (typeof window !== "undefined" ? window : {}).pdfjsWorker = workerjs;
  dpr.value = window.devicePixelRatio || 1;
  viewportHeight.value = window.innerHeight;
  setWidth();
  // src如果是字符串或属于Uint8Array的二进制处理数据，则进入渲染流程
  if (
    (typeof props.src === "string" && props.src.length > 0) ||
    props.src instanceof Uint8Array
  ) {
    // pdf文档获取
    await getDoc();
    // pdf渲染核心函数
    renderPDF();
    window.addEventListener("resize", renderPDFWithDebounce);
    isAddEvent.value = true;
  }

  // 判断src是否更改进行更新
  watch(
    () => props.src,
    (src: string | Uint8Array) => {
      if (
        (typeof src === "string" && src.length > 0) ||
        src instanceof Uint8Array
      ) {
        getDoc();
        renderPDF();
        if (!isAddEvent.value) {
          window.addEventListener("resize", renderPDFWithDebounce);
          isAddEvent.value = true;
        }
      }
    }
  );
});

// ！还没拆解
onUnmounted(() => {
  clearTimeout(timer);
  clearTimeout(scrollTimer);
  cancelAnimationFrame(animFrameId);
  isAddEvent.value &&
    window.removeEventListener("resize", renderPDFWithDebounce);
});

// 容器回到顶层功能
// --- back to top ---
let animFrameId: number;
const easeOutCubic = (progress: number) => {
  return 1 - Math.pow(1 - progress, 3);
};
const backToTop = () => {
  const duration = 500;
  const startPos = scroller.value.scrollTop;
  const startTime = performance.now();

  const animateScroll = (timestamp: number) => {
    const elapsed = timestamp - startTime;
    const progress = Math.min(elapsed / duration, 1);
    const easeProgress = easeOutCubic(progress);
    const distance = startPos * (1 - easeProgress);

    scroller.value.scrollTo(0, distance);

    if (progress < 1) {
      animFrameId = requestAnimationFrame(animateScroll);
    }
  };
  cancelAnimationFrame(animFrameId);
  requestAnimationFrame(animateScroll);
};
let waitToPageFun: Function | null = null;

watch(
  () => props.page,
  (page: number) => {
    if (props.page === currentPage.value) {
      return;
    }
    if (page > itemHeightList.value.length) {
      page = itemHeightList.value.length;
    }
    if (renderComplete.value) {
      scroller.value.scrollTo(0, (itemHeightList.value[page - 2] ?? 0) + 2);
    } else {
      waitToPageFun = () => {
        scroller.value.scrollTo(0, (itemHeightList.value[page - 2] ?? 0) + 2);
      };
    }
  }
);
watch(
  () => renderComplete.value,
  (complete: boolean) => {
    complete && waitToPageFun?.();
    waitToPageFun = null;
  }
);
watch(
  () => currentPage.value,
  (page: number) => {
    emit("onPageChange", page);
  }
);
</script>

<template>
  <div
    class="pdf-vue3-main"
    style="height: 100%; position: relative; min-height: 10px"
  >
    <div class="pdf-vue3-container" style="height: 100%">
      <div
        ref="scroller"
        class="pdf-vue3-scroller"
        style="height: 100%; overflow-y: auto"
        :style="{ maxHeight: `${viewportHeight}px` }"
        @scroll="handleScroll"
      >
        <div
          class="pdf-vue3-canvas-container"
          ref="container"
          style="margin: 0 auto"
          :style="{
            width: isNaN(Number(props.pdfWidth))
              ? props.pdfWidth
              : `${props.pdfWidth}px`,
          }"
        >
          <div
            style="position: relative; display: inline-block; width: 100%"
            v-for="(item, index) in canvasRefs"
            :key="index"
          >
            <canvas
              style="
                display: inline-block;
                box-shadow: #a9a9a9 0px 1px 3px 0px;
                margin-left: auto;
                margin-right: auto;
                width: calc(100% - 4px);
              "
              :style="{
                marginBottom: `${rowGap}px`,
              }"
              :ref="(instance: HTMLCanvasElement) => (canvasRefs[index] = instance)"
            ></canvas>
            <div
              class="annotationLayer"
              style="display: inline-block; width: 100%; height: 100%"
              :ref="(instance: HTMLElement) => (annotationLayerRefs[index] = instance)"
            ></div>
          </div>
        </div>
      </div>
    </div>
    <div
      class="pdf-vue3-progress"
      v-if="props.showProgress"
      style="
        position: absolute;
        left: 0;
        top: 0;
        width: 100%;
        user-select: none;
        pointer-events: none;
      "
    >
      <!-- 暴露给外界,进度条等可以自定义 -->
      <slot v-if="slots.progress" name="progress" :loadRatio="loadRatio"></slot>
      <!-- 如果没有外界的自定义进度条,则用默认 -->
      <div
        v-else
        style="width: 0%; height: 4px; transition: all 0.2s"
        :style="{
          width: `${loadRatio}%`,
          opacity: loadRatio < 100 ? '1' : '0',
          backgroundColor: props.progressColor,
        }"
      ></div>
    </div>
    <div
      class="pdf-vue3-pageTooltip"
      v-if="props.showPageTooltip"
      style="
        position: absolute;
        left: 12px;
        top: 12px;
        width: calc(100% - 12px);
        user-select: none;
        pointer-events: none;
      "
    >
      <slot
        v-if="slots.pageTooltip"
        name="pageTooltip"
        :currentPage="currentPage"
        :totalPages="totalPages"
      ></slot>
      <div
        v-else
        style="
          padding: 4px 8px;
          background: rgba(0, 0, 0, 0.5);
          color: #ffffff;
          font-size: 16px;
          border-radius: 6px;
          display: inline-block;
          transition: opacity 0.3s;
        "
        :style="{ opacity: isScrolling && totalPages > 0 ? '1' : '0' }"
      >
        {{ currentPage }}/{{ totalPages }}
      </div>
    </div>
    <div
      class="pdf-vue3-backToTopBtn"
      v-if="props.showBackToTopBtn"
      @click="backToTop"
      style="
        position: absolute;
        right: 16px;
        bottom: 16px;
        display: inline-block;
        user-select: none;
        pointer-events: none;
        opacity: 0;
        transition: opacity 0.3s;
      "
      :style="
        scrollOffset > props.scrollThreshold
          ? { opacity: '1', pointerEvents: 'initial' }
          : undefined
      "
    >
      <slot
        v-if="slots.backToTopBtn"
        name="backToTopBtn"
        :scrollOffset="scrollOffset"
      ></slot>
      <div
        v-else
        style="
          width: 50px;
          height: 50px;
          background: rgba(0, 0, 0, 0.4);
          color: #ffffff;
          font-size: 16px;
          border-radius: 50%;
          display: flex;
          justify-content: center;
          align-items: center;
        "
      >
        <svg
          width="24"
          height="24"
          viewBox="0 0 24 24"
          fill="none"
          xmlns="http://www.w3.org/2000/svg"
        >
          <path
            fill-rule="evenodd"
            clip-rule="evenodd"
            d="M11.0001 22.2877H13.0001V7.80237L16.2428 11.045L17.657 9.63079L12.0001 3.97394L6.34326 9.63079L7.75748 11.045L11.0001 7.80236V22.2877ZM18 3H6V1H18V3Z"
            fill="currentColor"
          />
        </svg>
      </div>
    </div>
  </div>
</template>

<style scoped>
  @import url(./index.css);
</style>
