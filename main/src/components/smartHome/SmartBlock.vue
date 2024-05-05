
<template>
	<div class="common-card">

		<div class="blur-background"></div>
		<div class="wuji-content _box is-flex is-flex-direction-column">
			<!-- Init State Start -->
			<h6 class="title is-4 mb-0 has-text-white is-flex-shrink-0">{{ $t(`Smart Agriculture`) }}</h6>
			<div class="is-flex is-align-items-center is-flex-grow-1 _notice-content-text">
				<div class="info ">
					<div class="des two-line is-size-14px">
						{{ $t(`Vioneta Agro Smart Agriculture Platform`) }}
					</div>
				</div>
				<b-image :src="require('@/assets/img/logo/casa-white.svg')" class="is-100x100"></b-image>
			</div>
			<div class="buttons is-flex is-flex-shrink-0 is-flex-direction-row-reverse">
				<b-button class="mb-0" rounded size="is-small" type="is-primary" @click="openSyncPanel">{{
					$t(actionText)
				}}
				</b-button>
			</div>
			<!-- Init State End -->

		</div>
	</div>
</template>

<script>
import events from '@/events/events';

export default {
	name: "sync-block",
	data() {
		return {
			isLoading: false,
			isStarting: false,
			syncBaseURL: "",
			isSyncInstalled: false,
			isSyncRunning: false,
			syncPort: "",
			syncId: ""
		}
	},
	created() {
		this.checkSyncStatus()

		this.$EventBus.$on(events.UPDATE_SYNC_STATUS, () => {
			this.checkSyncStatus();
		});

	},
	beforeDestroy() {
		this.$EventBus.$off(events.UPDATE_SYNC_STATUS);
	},
	computed: {
		actionText() {
			return !this.isSyncInstalled ? "Install" : "Open"
		}
	},

	methods: {
		async checkSyncStatus() {
			// const res = await this.$api.sys.getSystemApps()
			const listRes = await this.$api.container.getMyAppList();
			const systemApps = listRes.data ? listRes.data.data.casaos_apps : [];
			const is8384SyncInstalled = systemApps.some(app => {
				return app.image.includes('vionetaos') && app.port === 8384
			})
			if (is8384SyncInstalled) {
				this.isSyncInstalled = true
				this.syncBaseURL = `http://${this.$baseIp}:8123`
				this.syncPort = 8384
				this.syncId = systemApps.find(app => {
					return app.image.includes('vionetaos') && app.port === 8123
				}).port
				this.isSyncRunning = systemApps.some(app => {
					return app.image.includes('vionetaos') && app.port === 8123 && app.state === 'running'
				})
			} else {
				this.isSyncInstalled = systemApps.some(app => {
					return app.image.includes('vionetaos')
				})
				if (this.isSyncInstalled) {
					this.isSyncRunning = systemApps.some(app => {
						return app.image.includes('vionetaos') && app.state === "running"
					})
					this.syncPort = systemApps.find(app => {
						return app.image.includes('vionetaos')
					}).port
					this.syncId = systemApps.find(app => {
						return app.image.includes('vionetaos')
					}).id
					this.syncBaseURL = `http://${this.$baseIp}:${this.syncPort}`
				}
			}


		},
		async openSyncPanel() {
			await this.checkSyncStatus()
			if (!this.isSyncInstalled) {
				this.$EventBus.$emit(events.OPEN_APP_STORE_AND_GOTO_SYNCTHING);
			} else {
				if (this.isSyncRunning) {
					window.open(this.syncBaseURL, '_blank');
				} else {
					this.$buefy.dialog.confirm({
						title: ' ',
						message: this.$t('vioneta agro is not running, start it?'),
						hasIcon: true,
						closeOnConfirm: false,
						confirmText: this.$t('Start'),
						cancelText: this.$t('Cancel'),
						onConfirm: (value, { close }) => {
							this.$buefy.toast.open({
								message: this.$t(`Starting vioneta agro...`),
								type: 'is-white'
							})
							this.$api.container.updateState(this.syncId, "start").then((res) => {
								this.isStarting = false
								if (res.data.success == 200) {
									this.$EventBus.$emit(events.RELOAD_APP_LIST);
									setTimeout(() => {
										close()
										window.open(this.syncBaseURL, '_blank');

									}, 2000)
								} else {
									this.$buefy.toast.open({
										message: this.$t(`Failed to start, please try again.`),
										type: 'is-danger'
									})
								}
							})
						}
					})
				}
			}
		},

	},
	sockets: {
		"app:install-end"() {
			this.checkSyncStatus();
		},
		"app:install-error"() {
			this.checkSyncStatus();
		},
	}
}
</script>

<style lang="scss" scoped>
._box {
	height: 10rem;
	padding: 1rem 1.25rem;
	padding-top: 1.125rem;
	margin-bottom: 1rem;

	._notice-content-text {
		margin-right: 2rem;
		margin-left: 2rem;
	}
}

.des {
	line-height: 1.5em;
}
</style>
