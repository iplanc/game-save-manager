<template>
	<el-main>
		<el-card class="game-info">
			<div class="top-part">
				<img class="game-icon" :src="game_icon_src" />
			</div>
			<div style="padding: 14px" class="input-container">
				<div class="bottom">
					<el-button
						@click="choose_game_icon()"
						type="text"
						class="button"
						disabled
					>
						选择游戏图标(请使用方形图片)
					</el-button>
					<el-input v-model="game_name" placeholder="请输入游戏名（必须）" >
						<template #prepend>
							游戏名称
						</template>
					</el-input>
					<el-input v-model="save_path" placeholder="请选择存档路径（必须）">
						<template #prepend>
							存档目录
						</template>
						<template #append>
							<el-button @click="choose_save_directory()">
								<el-icon>
									<document-add />
								</el-icon>
							</el-button>
						</template>
					</el-input>
					<el-input v-model="game_path" placeholder="请选择游戏启动文件（非必须）">
						<template #prepend>
							启动文件
						</template>
						<template #append>
							<el-button @click="choose_executable_file()">
								<el-icon>
									<document-add />
								</el-icon>
							</el-button>
						</template>
					</el-input>
				</div>
			</div>
		</el-card>
	</el-main>
	<el-footer>
		<el-container class="submit-bar">
			<el-tooltip
				v-for="button in buttons"
				:key="button.id"
				:content="button.text"
				placement="top"
			>
				<el-button
					@click="submit_handler(button.method)"
					:type="button.type"
					circle
				>
					<el-icon>
						<component :is="button.icon" />
					</el-icon>
				</el-button>
			</el-tooltip>
		</el-container>
	</el-footer>
</template>

<script>
// !为了不让 submit_handler 报错，这里不使用 lang="ts"
import { defineComponent } from "vue";
import { DocumentAdd } from "@element-plus/icons-vue";
import { ElNotification } from "element-plus";
const { ipcRenderer } = require("electron");
import {
	Check,
	RefreshRight,
	Download,
	MagicStick,
} from "@element-plus/icons-vue";
import router from "@/router";

export default defineComponent({
	mounted() {
		ipcRenderer.on("reply_choose_save_directory", (Event, arg) => {
			// 选择游戏存档目录
			this.save_path = arg.filePaths[0] ? arg.filePaths[0] : this.save_path;
		});
		ipcRenderer.on("reply_choose_executable_file", (Event, arg) => {
			// 选择游戏存档目录
			this.game_path = arg.filePaths[0] ? arg.filePaths[0] : this.game_path;
		});
		ipcRenderer.on("reply_choose_game_icon", (Event, arg) => {
			// 选择游戏图标地址
			this.game_icon_src = arg;
		});
		ipcRenderer.on("reply_add_game", (Event, arg) => {
			if (arg) {
				ElNotification({
					title: "提示",
					message: "添加成功",
					type: "success",
					duration: 2000,
				});
			} else {
				ElNotification({
					title: "错误",
					message: "添加存档失败，请保证游戏名无重复且输入正确无误",
					type: "warning",
					duration: 2000,
				});
			}
			ipcRenderer.send("get_config");
		});
	},
	components: { DocumentAdd, Check, RefreshRight, Download, MagicStick },
	data() {
		return {
			game_name: "", // 写入游戏名
			save_path: "", // 选择游戏存档目录
			game_path: "", // 选择游戏启动程序
			game_icon_src:
				"https://shadow.elemecdn.com/app/element/hamburger.9cf7b091-55e9-11e9-a976-7f4d0b07eef6.png",
			buttons: [
				{
					text: "修改已保存的配置",
					type: null,
					icon: "MagicStick",
					method: "change",
				},
				{
					text: "自动识别本地游戏",
					type: "primary",
					icon: "Download",
					method: "search_local",
				},
				{
					text: "保存当前编辑的配置",
					type: "success",
					icon: "Check",
					method: "save",
				},
				{
					text: "重置当前配置",
					type: "danger",
					icon: "RefreshRight",
					method: "reset",
				},
			],
		};
	},
	methods: {
		choose_save_directory() {
			// 选择游戏存档目录
			ipcRenderer.send("choose_save_directory");
		},
		choose_executable_file() {
			// 选择游戏启动程序
			ipcRenderer.send("choose_executable_file");
		},
		choose_game_icon() {
			// 选择游戏图标地址
			// TODO:需要优化图片选择，自由切割和压缩到指定大小
			ipcRenderer.send("choose_game_icon");
		},
		submit_handler(button_method) {
			// 映射按钮的ID和他们要触发的方法
			this[button_method]();
		},
		search_local() {
			// TODO:导入已有配置
			ElNotification({
				type: "warning",
				message: "--WIP-- 这个功能尚未完成",
			});
		},
		change() {
			router.push("/change-game-info");
		},
		save() {
			if(this.game_name==""||this.save_path==""){
				ElNotification({
					type:"error",
					message:"请至少输入游戏名和存档路径"
				})
				return
			}
			// 保存当前配置
			console.log("保存当前编辑的配置");
			ipcRenderer.send("add_game", {
				game_name: this.game_name,
				save_path: this.save_path,
				icon: this.game_icon_src,
				game_path: this.game_path,
			});
		},
		reset() {
			// 重置当前配置
			this.game_name = "";
			this.save_path = "";
			this.game_path = "";
			this.game_icon_src =
				"https://shadow.elemecdn.com/app/element/hamburger.9cf7b091-55e9-11e9-a976-7f4d0b07eef6.png";
			ElNotification({
				title: "提示",
				message: "重置成功",
				type: "success",
				duration: 1000,
			});
		},
	},
	beforeUnmount() {
		ipcRenderer.removeAllListeners("reply_choose_save_directory");
		ipcRenderer.removeAllListeners("reply_choose_executable_file");
		ipcRenderer.removeAllListeners("reply_choose_game_icon");
		ipcRenderer.removeAllListeners("reply_add_game");
	},
});
</script>

<style>
.game-info {
	height: 99%;
}
.top-part {
	height: 200px;
}
.game-icon {
	float: left;
	height: 200px;
	width: 200px;
}
.el-input {
	margin-top: 10px;
}
.submit-bar {
	justify-content: flex-end;
}
</style>