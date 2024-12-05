<template>
	<div class="container">
		<h2>PDF 文件上传与权限校验</h2>

		<el-upload
			ref="upload"
			:action="`${api}/upload`"
			accept="application/pdf"
			:before-upload="beforeUpload"
			:on-success="onSuccess"
			:on-exceed="handleExceed"
		>
			<el-icon class="el-icon--upload"><upload-filled /></el-icon>
			<div class="el-upload__text">
				<em>点击上传</em>
			</div>
		</el-upload>


		<!-- 错误信息 -->
		<div v-if="error" class="error">{{ error }}</div>

		<!-- 未加密文件信息 -->
		<div v-if="fileInfo && !fileInfo.isEncrypted" class="info-card">
			<p>文件未加密</p>
			<p>总页数：{{ fileInfo.pageCount }}</p>
		</div>

		<!-- 加密文件弹窗 -->
		<el-dialog title="输入密码解锁 PDF 文件" v-model="showPasswordDialog" width="30%" @open="openPasswordDialog">
			<el-form @submit.native.prevent>
				<el-form-item label="密码">
					<el-input ref="inputPasswordRef" v-model="password" type="password" placeholder="请输入密码" @keyup.enter.native="unlockPDF" />
				</el-form-item>
				<el-form-item>
					<el-button type="primary" @click="unlockPDF">解锁</el-button>
					<el-button @click="cancelUnlock">取消</el-button>
				</el-form-item>
			</el-form>
		</el-dialog>

		<div v-if="fileInfo?.isEncrypted && fileInfo.pageCount">
			<p>总页数：{{ fileInfo.pageCount }}</p>
			<p>密码：{{ password }}</p>
		</div>
	</div>
</template>

<script setup>
import { UploadFilled  } from "@element-plus/icons-vue";
import { ref } from "vue";
import { ElMessage, genFileId } from "element-plus";
import { getDocument } from "pdfjs-dist";
import * as pdfjsLib from "pdfjs-dist";
// pdfjsLib.GlobalWorkerOptions.workerSrc = new URL("pdfjs-dist/build/pdf.worker.min.mjs", import.meta.url).href;
pdfjsLib.GlobalWorkerOptions.workerSrc = "http://unpkg.com/pdfjs-dist@4.9.124/legacy/build/pdf.worker.min.mjs"

const api = "http://8.129.13.2:3000";
// 状态管理
const error = ref(null); // 错误信息
const fileInfo = ref(null); // 文件信息
const password = ref(""); // 用户输入的密码
const showPasswordDialog = ref(false); // 控制弹窗显示
const upload = ref(null);
const inputPasswordRef = ref(null);
const beforeUpload = file => {
	const isPDF = file.type === "application/pdf";
	if (!isPDF) {
		ElMessage.error("只能上传 PDF 文件");
		return false;
	}
};

const onSuccess = async (response, uploadFile, uploadFiles) => {
	console.log(response);
	console.log(uploadFile);
	try {
		// 尝试加载 PDF
		console.log(uploadFile.url);
		const loadingTask = getDocument({ url: api + response.file });
		const pdfDoc = await loadingTask.promise;

		console.log("PDF 文件未加密，页数:", pdfDoc.numPages);

		// 未加密文件处理
		fileInfo.value = {
			isEncrypted: false,
			pageCount: pdfDoc.numPages
		};
		error.value = null;
	} catch (err) {
		console.error("PDF 加载错误:", err);

		// 检测加密文件
		if (err.name === "PasswordException") {
			console.log("文件已加密");
			fileInfo.value = {
				isEncrypted: true,
				url: api + response.file,
				pageCount: null
			};
			showPasswordDialog.value = true;
			error.value = null;
		} else {
			error.value = "加载 PDF 文件失败：" + err.message;
		}
	}
};

const handleExceed = files => {
	upload.value.clearFiles();
	const file = files[0];
	file.uid = genFileId();
	upload.value.handleStart(file);
};

// 处理文件上传
async function handleFileUpload(event) {
	console.log(event);
	const file = event.file; // 获取用户上传的文件
	// console.log(file);
	if (!file) {
		error.value = "未选择文件";
		return;
	}

	console.log("上传的文件:", file);

	const fileReader = new FileReader();
	fileReader.onload = async () => {
		console.log("文件读取成功");
		const pdfData = new Uint8Array(fileReader.result);
		console.log(pdfData);
		try {
			// 尝试加载 PDF
			const loadingTask = getDocument({ data: pdfData });
			const pdfDoc = await loadingTask.promise;

			console.log("PDF 文件未加密，页数:", pdfDoc.numPages);

			// 未加密文件处理
			fileInfo.value = {
				isEncrypted: false,
				pageCount: pdfDoc.numPages
			};
			error.value = null;
		} catch (err) {
			console.error("PDF 加载错误:", err);

			// 检测加密文件
			if (err.name === "PasswordException") {
				console.log("文件已加密");
				fileInfo.value = {
					isEncrypted: true,
					pdfData,
					pageCount: null
				};
				showPasswordDialog.value = true;
				error.value = null;
			} else {
				error.value = "加载 PDF 文件失败：" + err.message;
			}
		}
	};

	fileReader.onerror = () => {
		console.error("文件读取失败:", fileReader.error);
		error.value = "文件读取失败，请重试。";
	};

	fileReader.readAsArrayBuffer(file); // 读取文件
}

// 解锁加密的 PDF
async function unlockPDF() {
	if (!password.value) {
		ElMessage.error("请输入密码");
		return;
	}
	try {
		console.log("密码:", password.value);
		const loadingTask = getDocument({
			// data: fileInfo.value.pdfData,
			url: fileInfo.value.url,
			password: password.value
		});
		const pdfDoc = await loadingTask.promise;

		console.log("解锁成功，页数:", pdfDoc.numPages);

		// 成功解锁
		fileInfo.value.pageCount = pdfDoc.numPages;
		error.value = ''
		showPasswordDialog.value = false;
		ElMessage.success("解锁成功！");
	} catch (err) {
		console.error("解锁失败:", err);
		error.value = "密码错误，请重试";
		ElMessage.error(error.value);
	}
}

// 取消解锁
function cancelUnlock() {
	password.value = "";
	showPasswordDialog.value = false;
}

const openPasswordDialog = () => {
	password.value = "";
	setTimeout(() => {
		inputPasswordRef.value.focus()
	}, 0);
}

</script>

<style scoped lang="less">
.container{
	width: 430px;
}
h2{
	margin-bottom: 20px;
	text-align: center;
}
.error {
	color: red;
	margin-top: 10px;
}
.info-card {
	margin-top: 20px;
}
:deep(.el-upload) {
  border: 1px dashed #dcdfe6;
  border-radius: 6px;
  cursor: pointer;
  position: relative;
  overflow: hidden;
  transition: var(--el-transition-duration-fast);
  padding: 50px;
  flex-direction: column;
  width: 100%;
}

.el-upload:hover {
  border-color: var(--el-color-primary);
}

.el-icon.avatar-uploader-icon {
  font-size: 28px;
  color: #8c939d;
  width: 178px;
  height: 178px;
  text-align: center;
}
.el-icon.el-icon--upload{
	font-size: 64px;
}
</style>
