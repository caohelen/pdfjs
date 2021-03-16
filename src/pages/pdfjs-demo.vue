<!--查看pdf票据-->
<template>
  <div>
    <input type="file" ref="fielinput" @change="uploadFile" />
		<button @click="getContent">获取信息</button>
    <div id="pdf-container">
		</div>
		<!-- <div class="shadePosition"></div> -->
  </div>
</template>

<script>
  import pdfJS from 'pdfjs-dist'
	import { TextLayerBuilder } from 'pdfjs-dist/web/pdf_viewer';
	import 'pdfjs-dist/web/pdf_viewer.css';
  export default {
    mounted() {
    },
    data() {
      return {
        pdfData: '', // PDF的base64
        scale: 1, // 缩放值
				container: '',
				pageDiv: '',
				textLayer:'',
				arcX: null,
				arcY: null,
				count: 0, // 可以理解为计算行数使用
				startTop: '',
				startSelectText: '', // 开始节点数据
				endTop: '',
				endSelectText: '',
      }
    },
    methods: {
			getContent() {
				// console.log(window.getSelection().toString());
			},
			getPDF() {
				// 引入pdf.js的字体
        let CMAP_URL = 'https://unpkg.com/pdfjs-dist@2.0.943/cmaps/'
        //读取base64的pdf流文件
					pdfJS.getDocument({
          data: this.pdfData, // PDF base64编码
          cMapUrl: CMAP_URL,
          cMapPacked: true
        }).then((pdf) => {
							this.container = document.getElementById('pdf-container');
							for (var i = 1; i<= pdf.numPages; i++) {
									this.renderPDF(pdf, i);
							}
							
					})
			},
			renderPDF(pdf, num) {
					pdf.getPage(num).then((page) => {
							var scale = 1.5;
							var viewport = page.getViewport(scale);
							var pageDiv = document.createElement('div');
							pageDiv.setAttribute('id', 'page-' + (page.pageIndex + 1));
							pageDiv.setAttribute('style', 'position: relative');
							this.container.appendChild(pageDiv);
							var canvas = document.createElement('canvas');
							pageDiv.appendChild(canvas);
							var context = canvas.getContext('2d');
							canvas.height = viewport.height;
							canvas.width = viewport.width;
							
							var renderContext = {
									canvasContext: context,
									viewport: viewport
							};
							
							page.render(renderContext)
							.then(() => {
									return page.getTextContent();
							}).then((textContent) => {
									// 创建文本图层div
									const textLayerDiv = document.createElement('div');
									// 获取canvas宽高
									const canvas = document.getElementsByTagName('canvas')
									const textLayerDivWidth = canvas[0].width
									const textLayerDivHeight = canvas[0].height
									textLayerDiv.setAttribute('class', 'textLayer');
									// 宽高设置成canvas一样
									textLayerDiv.setAttribute('style', `width:${textLayerDivWidth}px;height:${textLayerDivHeight}px`);
									// 将文本图层div添加至每页pdf的div中
									pageDiv.appendChild(textLayerDiv);

									// 创建新的TextLayerBuilder实例
									var textLayer = new TextLayerBuilder({
											textLayerDiv: textLayerDiv,
											pageIndex: page.pageIndex,
											viewport: viewport
									});
									textLayer.setTextContent(textContent);
									textLayer.render();
									textLayerDiv.addEventListener('mousedown', (evt) => {
										
											// eslint-disable-next-line no-console
											// evt = window.event || evt;
											// //获取圆心的位置		
											// //获取鼠标相对于文档的位置
											const documentWidth = document.body.clientWidth
											const leaveNoteWidth = (documentWidth - textLayerDivWidth) / 2
											this.arcX = evt.pageX - leaveNoteWidth - 9;
											// this.arcY = evt.pageY || evt.clientY+scrollY;
										})
									// this.$nextTick(() => {
											
											
										textLayerDiv.addEventListener('mouseup', () => {
												this.textLayer = document.querySelector('.textLayer')
												var sel = window.getSelection();
												if(sel.toString() !== '') {
												// console.log(sel);
													let textLayerChilds = this.textLayer.children
													let startN = null // 用于定位首个节点的位置
													let endN = null // 用于定位最后一个节点的位置
													for(let n in textLayerChilds) {
														// 找到选取的首个节点
														if(textLayerChilds[n] === sel.anchorNode.parentElement) {
															startN = Number(n)
															this.startTop = this.textLayer.children[startN].style.top.replace('px', '');
															// var nextTop = this.textLayer.children[startN + 1].style.top.replace('px', '');
															this.startSelectText = this.textLayer.children[startN].innerHTML.substr(sel.anchorOffset);
														}
														if(textLayerChilds[n] === sel.focusNode.parentElement) {
															// 找到选取的最后一个节点
															endN = Number(n)
															this.endTop = this.textLayer.children[endN].style.top.replace('px', '')
															this.endSelectText = this.textLayer.children[endN].innerHTML.substr(0, sel.focusOffset)
														}
													}
													this.applyColours(startN, endN)
												}
											})
										});
							// });
					});
			},
			// 渲染中间行
			applyColours(startN, endN) {
				const sel = window.getSelection();
				if(sel.toString() !== '') {
					let startSelectText = ''
					if(startN === endN) { // 选中一行的情况并且一行中只有一个节点
						const str = window.getSelection().toString()
						const top = this.textLayer.children[startN].style.top.replace('px', '')
						const sizeObj =  this.getSize(str, startN)
						this.createdOverlayDiv(sizeObj, top, this.arcX);
						return;
					}
					if(this.count > 0) { // 如果换行了,获取不同的text
						startSelectText = this.textLayer.children[startN].innerHTML
					} else {
						startSelectText = this.textLayer.children[startN].innerHTML.substr(sel.anchorOffset);
					}
					for(let i = startN + 1; i <= endN; i++) {
						const startTop = this.textLayer.children[startN].style.top.replace('px', '')
						const nextTop = this.textLayer.children[i].style.top.replace('px', '')
						const nextLeft = this.textLayer.children[startN].style.left.replace('px', '')
						let nextText = ''
						if(Math.abs(startTop - nextTop) <= 10) { // 第一个节点与下一个节点在同一行
							if(this.textLayer.children[i] === sel.focusNode.parentElement) { // 判断是否是最后一个节点
								nextText = this.textLayer.children[i].innerHTML.substr(0, sel.focusOffset)
							} else {
								nextText = this.textLayer.children[i].innerHTML
							}
							startSelectText = startSelectText + nextText
							const sizeObj =  this.getSize(startSelectText, endN)
							if(this.count > 0) {
								this.createdOverlayDiv(sizeObj, startTop, nextLeft);
							} else {
								this.createdOverlayDiv(sizeObj, startTop, this.arcX);
							}
						} else {
							if(this.count === 0) {
								const isShadePosition = document.querySelector('.shadePosition')
								if(isShadePosition === null) { // 判断是否渲染过内容,如果有就不必要执行以下渲染
									const sizeObj =  this.getSize(this.startSelectText, startN);
									this.createdOverlayDiv(sizeObj, this.startTop, this.arcX);
								}
								const next2Top = this.textLayer.children[i + 1].style.top.replace('px', '')
								if(Math.abs(nextTop - next2Top) > 10) { // 说明开始节点的后面两个节点还是单行的
									this.count = 0;
									this.otherMothodload(i, endN);
									return;
								} else {
									this.count++;
									this.applyColours(i, endN);
									return;
								}
							} else {
								this.count++;
								this.applyColours(i, endN)
							}
						}
						if(i === endN) {
							this.count = 0
							sel.empty()
							return;
						}
					}
				}
			},
			otherMothodload(start, endN) {
				const sel = window.getSelection();
				for(let n = start; n <= endN; n++) {
					const Top = this.textLayer.children[n].style.top.replace('px', '')
					const left = this.textLayer.children[n].style.left.replace('px', '')
					const nextTop = this.textLayer.children[n + 1].style.top.replace('px', '')
					let text = ''
					if(Math.abs(Top - nextTop) <= 10) {
						const nextText = this.textLayer.children[n + 1].innerHTML
						if(sel.focusNode !== null) {
							if(this.textLayer.children[n] === sel.focusNode.parentElement) {
								text = this.textLayer.children[n].innerHTML.substr(0, sel.focusOffset)
								sel.empty()
							} else {
								text = this.textLayer.children[n].innerHTML + nextText
							}
						}
						const sizeObj =  this.getSize(text, n);
						this.createdOverlayDiv(sizeObj, Top, left);
						this.otherMothodload(n + 2, endN)
					} else {
						if(sel.focusNode !== null) {
							if(this.textLayer.children[n] === sel.focusNode.parentElement) {
								text = this.textLayer.children[n].innerHTML.substr(0, sel.focusOffset)
								sel.empty()
							} else {
								text = this.textLayer.children[n].innerHTML
							}
						}
						const sizeObj =  this.getSize(text, n);
						this.createdOverlayDiv(sizeObj, Top, left);
					}
				}
			},

			computedRange(num, mytxt) {
				//获取文本框内当前光标的位置
				let range = document.getElementById(mytxt).createTextRange();
				let rect = range.getClientRects();//返回一个矩形
				let left = rect[0].left;
				if(num > rect.length - 1 || num < 0)
					return;
				if(num == 0) //选择第一行的情况
				{
						//设置选择范围
						let right = rect[0].right;
						range.moveEnd("character",-range.text.length);
						while(range.offsetLeft + range.boundingWidth < right)
						{
						range.expand("character");
						}
						return range;
				}
				else
				{
					//设置选择范围
					let right = rect[num].right;
					let range = this.computedRange(num - 1, mytxt);
					range.moveStart("character",range.text.length + 1);
					while((range.offsetLeft + range.boundingWidth) < right)
					{
					range.expand("character");
					}
					if(range.offsetLeft > left)
					range.moveStart("character",-1);
					return range;
				}
			},
		//选择指定行数的方法
     getText(num){
			var range = this.computedRange(num,"txt")//调用真正的获取行方法
			if(range != null) //如果指定的行内容不为空
				{
					alert(range.text);
					range.select(); //选择指定的行
				}
			},

			getSize(str, n) {
				const fontSize = this.getFontSize(n)
				const fontFamily = this.getFontFamily(n)
				const sizeObj = this.computeFontSize(str, fontSize, fontFamily);
				return sizeObj;
			},

			getInnerHTML(n) {
				return this.textLayer.children[n].innerHTML
			},
			getFontSize(n) {
				return this.textLayer.children[n].style.fontSize
			},
			getFontFamily(n) {
				return this.textLayer.children[n].style.fontFamily
			},
			// 创建遮罩层
			createdOverlayDiv(obj, Top, Left) {
				const width = obj.width
				const shadePositionDiv = document.createElement('div');
				shadePositionDiv.setAttribute('class', 'shadePosition');
				shadePositionDiv.setAttribute('style', `width: ${width}px; height: ${obj.height}px; left: ${Left}px; top: ${Top}px`);
				this.textLayer.appendChild(shadePositionDiv)
			},

			computeFontSize(str, size, family) {
				let spanDom = document.createElement("span");
				spanDom.style.fontSize = size;
				spanDom.style.opacity = "0";
				spanDom.style.fontFamily = family;
				spanDom.innerHTML = str;
				document.body.append(spanDom);
				let sizeD = {};
				sizeD.width = spanDom.offsetWidth;
				sizeD.height = spanDom.offsetHeight;
				spanDom.remove();
				return sizeD;
			},

			// var canvas = document.createElement("canvas");
			// var ctx = canvas.getContext("2d");
			// ctx.font = textLayer.children[startN].style.fontFamily;  // This can be set programmaticly from the element's font-style if desired
			// var textHeight = ctx.measureText(text).width;

			// if (sel.rangeCount) {
			// 	const range = sel.getRangeAt(0).cloneRange();
			// 	console.log(range.getBoundingClientRect());
			// 	// if (range.getBoundingClientRect) {
			// 	// 		var rect = range.getBoundingClientRect();
			// 	// 		var width = rect.right - rect.left;
			// 	// 		var height = rect.bottom - rect.top;
			// 	// }
			// }

			
      uploadFile () {
        let inputDom = this.$refs.fielinput
        let file = inputDom.files[0]
        let reader = new FileReader()
        reader.readAsDataURL(file)
        reader.onload = () => {
          this.pdfData = atob(reader.result.substring(reader.result.indexOf(',') + 1))
          // this.previewPDF()
					this.getPDF()
        }
      },
      previewPDF() {
        // 引入pdf.js的字体
        let CMAP_URL = 'https://unpkg.com/pdfjs-dist@2.0.943/cmaps/'
        //读取base64的pdf流文件
        let loadingTask = pdfJS.getDocument({
          data: this.pdfData, // PDF base64编码
          cMapUrl: CMAP_URL,
          cMapPacked: true
        })
        loadingTask.promise.then((pdf) => {
          this.loadFinished = true
          let numPages = pdf.numPages
          let pageNumber = 1
          this.getPage(pdf, pageNumber, numPages)
        })
      },
      getPage(pdf, pageNumber, numPages) {
        let _this = this
        pdf.getPage(pageNumber)
          .then((page) => {
            // 获取DOM中为预览PDF准备好的canvasDOM对象
            let canvas = this.$refs.myCanvas
            let viewport = page.getViewport(_this.scale)
            canvas.height = viewport.height
            canvas.width = viewport.width

            let ctx = canvas.getContext('2d')
            let renderContext = {
              canvasContext: ctx,
              viewport: viewport
            }
            page.render(renderContext).then(() => {
              pageNumber += 1
              if (pageNumber <= numPages) {
                _this.getPage(pdf, pageNumber, numPages)
              }
            })
          })
      }
    }
  }
</script>

<style>
  .pdf-container {
    width: 750px;
  }
  .canvas-container {
    border: 1px dashed black;
    position: relative;
  }
	.shadePosition {
		background: #000;
	}
	.textLayer{
		left: 50%;
		transform: translateX(-50%);
	}
</style>
