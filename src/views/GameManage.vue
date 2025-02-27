<!--
 * @Author: PlanC
 * @Date: 2022-09-28 12:13:50
 * @LastEditTime: 2022-09-28 12:47:58
 * @FilePath: \game-save-manager\src\views\GameManage.vue
-->

<template>
	<div class="manage-container">
		<!-- 下面是顶栏部分 -->
		<el-card class="manage-top-bar">
			<template v-for="button in top_buttons" :key="button.id">
				<el-button type="primary" round @click="button_handler(button.method)">
					{{ button.text }}
				</el-button>
			</template>
			<el-button type="danger" round @click="del_cur()">
				删除该存档管理
			</el-button>

			<!-- 下面是当前存档描述信息 -->

			<el-input v-model="describe" placeholder="请输入新存档描述信息">
				<template #prepend>{{ this.game.name }}的新存档: </template>
			</el-input>
		</el-card>
		<!-- 下面是主体部分 -->
		<el-card class="saves-container">
			<!-- 存档应当用点击展开+内部表格的方式来展示 -->
			<!-- 这里应该有添加新存档按钮，按下后选择标题和描述进行存档 -->
			<!-- 下面是测试用数据，最后需要被替换成v-for生成的时间轴卡片 -->
			<el-table :data="filter_table" style="width: 100%">
				<el-table-column label="备份日期" prop="date" width="200px" sortable />
				<el-table-column label="描述" prop="describe" />
				<el-table-column align="right">
					<template #header>
						<!-- 搜索 -->
						<el-input
							v-model="search"
							size="small"
							placeholder="输入以搜索描述"
							clearable
						/>
					</template>
					<template #default="scope">
						<!-- scope.$index和scope.row可以被使用 -->
						<el-popconfirm
							title="确定覆盖现有存档?"
							@confirm="apply_save(scope.row.date)"
						>
							<template #reference>
								<el-button size="small"> 应用 </el-button>
							</template>
						</el-popconfirm>
						<el-popconfirm
							title="确定要删除?"
							@confirm="del_save(scope.row.date)"
						>
							<template #reference>
								<el-button size="small" type="danger">
									删除
								</el-button></template
							>
						</el-popconfirm>
					</template>
				</el-table-column>
			</el-table>
		</el-card>
	</div>
</template>

<script>
// !这里不使用lang = 'ts'是为了保证button_handler运行
import { defineComponent } from "vue";
import { ElMessageBox, ElNotification } from "element-plus";
import { ipcRenderer } from "electron";
import { store } from "@/store";
import path from "path";

export default defineComponent({
	components: {},
	data() {
		return {
			top_buttons: [
				{ text: "创建新存档", method: "create_new_save" },
				{ text: "用最新存档覆盖", method: "load_latest_save" },
				{ text: "启动游戏", method: "launch_game" },
				{ text: "打开存档文件夹", method: "open_save_folder"},
			],
			search: "",
			table_data: [
				{
					date: "",
					describe: "这是一条错误信息，正常情况不会出现",
					path: "",
				},
			],
			game: {
				name: "",
				backup_path: "", // 原save_path，现改为backup_path，备份存档位置
				save_path: "", // 游戏存档位置
				game_path: "",
				icon: "",
			},
			describe: "",
			backup_button_time_limit: true, // 两次备份时间间隔1秒
			backup_button_backup_limit: true, // 上次没备份好禁止再备份或读取
			apply_button_apply_limit: true, // 上次未恢复好禁止读取或备份
		};
	},
	mounted() {
		ipcRenderer.on("reply_get_game_backup", (Event, arg) => {
			this.load_game(arg);
		});
		ipcRenderer.on("reply_delete_game", (Event, arg) => {
			if (!arg) {
				ElNotification({
					type: "warning",
					message: "删除失败",
				});
				return;
			}
			ElNotification({
				type: "info",
				message: "您删除了该存档管理",
			});
			ipcRenderer.send("get_config");
			this.$router.push("/home");
		});
		ipcRenderer.on("reply_backup", (Event, arg) => {
			let type;
			let message;
			console.log("备份结果:", arg);
			if (arg) {
				type = "success";
				message = "备份成功";
			} else {
				type = "error";
				message = "备份失败";
			}
			ElNotification({
				type: type,
				message: message,
			});
			this.backup_button_backup_limit = true;
			ipcRenderer.send("get_game_backup", {
				game_name: this.$route.params.name,
			});
		});
		ipcRenderer.on("reply_delete_save", (Event, arg) => {
			let type;
			let message;
			if (arg) {
				type = "success";
				message = "删除成功";
			} else {
				type = "error";
				message = "删除失败";
			}
			ElNotification({
				type: type,
				message: message,
			});
			ipcRenderer.send("get_game_backup", {
				game_name: this.$route.params.name,
			});
		});
		ipcRenderer.on("reply_apply_backup", (Event, arg) => {
			let type;
			let message;
			if (arg) {
				type = "success";
				message = "恢复成功";
			} else {
				type = "error";
				message = "恢复失败";
			}
			ElNotification({
				type: type,
				message: message,
			});
			this.apply_button_apply_limit = true;
		});
		ipcRenderer.on("reply_apply_backup_with_extra_backup", (Event, arg) => {
			if (!arg[0]) {
				throw "在reply_apply_backup_with_extra_backup中发生未知错误";
			}
			ElNotification({ message: "已经创建额外备份", type: "success" });
			ipcRenderer.send("apply_backup", {
				game_name: arg[1].game_name,
				save_date: arg[1].save_date,
			});
		});

		ipcRenderer.send("get_game_backup", {
			game_name: this.$route.params.name,
		});
	},
	beforeUnmount() {
		ipcRenderer.removeAllListeners("reply_get_game_backup");
		ipcRenderer.removeAllListeners("reply_delete_game");
		ipcRenderer.removeAllListeners("reply_backup");
		ipcRenderer.removeAllListeners("reply_delete_save");
		ipcRenderer.removeAllListeners("reply_apply_backup");
		ipcRenderer.removeAllListeners("reply_apply_backup_with_extra_backup");
	},
	methods: {
		load_game(saves) {
			// 在路由切换后，把当前游戏的信息读取到data的table_data中
			this.game.name = saves.name;
			this.table_data = saves.saves;
			this.game.backup_path = path.join(
				store.state.config.backup_path,
				this.game.name
			);
			this.game.save_path = store.getters.getConfig.games[this.game.name].save_path;
			this.game.game_path = store.state.config.games[this.game.name].game_path;
			this.game.icon = saves.icon;
		},
		button_handler(func) {
			// 触发按钮绑定的方法
			this[func]();
		},
		create_new_save() {
			// 先检查配置中的prompt_when_not_described（是否允许不输入描述就存档）
			if (
				store.state.config.settings.prompt_when_not_described &&
				!this.describe
			) {
				ElMessageBox.confirm("你没有给这个存档提供描述，继续吗？", "警告", {
					confirmButtonText: "坚持保存",
					cancelButtonText: "取消",
					type: "warning",
				})
					.then(() => {
						this.send_save_to_background();
					})
					.catch(() => {});
			} else {
				this.send_save_to_background();
			}
		},
		send_save_to_background() {
			ElNotification({
				type: "info",
				message: "当该游戏存档大时操作会很久，请等提示成功后再进行其他操作",
			});
			if (!this.backup_button_time_limit) {
				ElNotification({
					type: "error",
					message: "无法在一秒内进行多次存档",
				});
				return;
			}
			if (!this.backup_button_backup_limit) {
				ElNotification({
					type: "error",
					message: "上次备份还未完成，请等待",
				});
				return;
			}
			if (!this.apply_button_apply_limit) {
				ElNotification({
					type: "warning",
					message: "上次覆盖还未完成，请等待",
				});
				return;
			}
			ipcRenderer.send("backup", {
				game_name: this.game.name,
				describe: this.describe,
			});
			this.describe == "";
			this.backup_button_time_limit = false;
			this.backup_button_backup_limit = false;
			let that = this;
			setTimeout(() => {
				that.backup_button_time_limit = true;
			}, 1000);
		},
		load_latest_save() {
			if (this.table_data[0].date) {
				this.apply_save(this.table_data[0].date);
			} else {
				ElNotification({
					type: "error",
					message: "发生了错误，可能您没有任何存档",
				});
			}
		},
		launch_game() {
			if (this.game.game_path == undefined || this.game.game_path.length < 4) {
				ElNotification({
					type: "error",
					message: "您并没有储存过该游戏的启动方式",
				});
				return;
			} else {
				ipcRenderer.send("open_exe", this.game.game_path);
			}
		},
		open_save_folder() {
			// 初始化node-cmd实例
			var cmd = require("node-cmd");
			// debug输出
			console.log(this.game.save_path);
			// 打开存档目录
			cmd.run("explorer " + this.game.save_path);
		},
		del_save(date) {
			ipcRenderer.send("delete_save", {
				game_name: this.game.name,
				save_date: date,
			});
		},
		apply_save(date) {
			ElNotification({
				type: "info",
				message: "当该游戏存档大时操作会很久，请等提示成功后再进行其他操作",
			});

			if (!this.apply_button_apply_limit) {
				ElNotification({
					type: "warning",
					message: "上次覆盖还未完成，请等待",
				});
				return;
			}
			if (!this.backup_button_backup_limit) {
				ElNotification({
					type: "error",
					message: "上次备份还未完成，请等待",
				});
				return;
			}
			if (store.state.config.settings.extra_backup_when_apply) {
				ipcRenderer.send("apply_backup_with_extra_backup", {
					game_name: this.game.name,
					save_date: date,
				});
			} else {
				ipcRenderer.send("apply_backup", {
					game_name: this.game.name,
					save_date: date,
				});
			}
			this.apply_button_apply_limit = false;
		},
		del_cur() {
			ElMessageBox.prompt(
				"如果确定删除的话，请输入yes，否则请点击取消。这个操作将会抹除已经备份过的该游戏的所有存档，并且把该游戏从已识别列表中去除",
				"提示",
				{
					confirmButtonText: "确定",
					cancelButtonText: "取消",
					inputPattern: /yes/,
					inputErrorMessage: "无效的输入",
				}
			)
				.then(() => {
					ipcRenderer.send("delete_game", { game_name: this.game.name });
				})
				.catch(() => {
					ElNotification({
						type: "info",
						message: "您取消了这次操作",
					});
				});
		},
	},
	computed: {
		filter_table() {
			return this.table_data.filter(
				(data) =>
					!this.search ||
					data.describe.includes(this.search) ||
					data.date.includes(this.search)
			);
		},
	},
	created() {
		this.$watch(
			// TODO:需要根据路由来切换游戏
			() => this.$route.params,
			(newVal, oldVal) => {
				console.log("选中游戏", newVal.name);
				if (!newVal.name) {
					return;
				}
				ipcRenderer.send("get_game_backup", {
					game_name: newVal.name,
				});
			}
		);
	},
});
</script>

<style>
.manage-top-bar {
	width: 98%;
	padding-right: 10px;
	padding-left: 10px;
	margin: auto;
	margin-bottom: 5px;

	display: flex;
	border-radius: 10px;
	align-items: center;
	color: aliceblue;
}
.saves-container {
	margin: auto;
}
</style>
