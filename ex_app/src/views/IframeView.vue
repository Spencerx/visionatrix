<template>
	<div class="iframe-ui-viewer">
		<iframe
			v-show="!error && !loading"
			id="visionatrix-iframe"
			ref="iframe"
			:name="'visionatrix-iframe'"
			class="visionatrix__iframe"
			allow="clipboard-read *; clipboard-write *"
			:src="iframeSrc" />
		<NcLoadingIcon v-if="loading" :size="48" />
		<NcEmptyContent
			v-if="error && !loading"
			:name="t('visionatrix', 'Failed to load service iframe')"
			:description="t('visionatrix', 'Please try again.')">
			<template #icon>
				<AlertCircleIcon :size="20" />
			</template>
		</NcEmptyContent>
	</div>
</template>

<script>
import { generateUrl } from '@nextcloud/router'

import NcEmptyContent from '@nextcloud/vue/dist/Components/NcEmptyContent.js'
import AlertCircleIcon from 'vue-material-design-icons/AlertCircle.vue'
import NcLoadingIcon from '@nextcloud/vue/dist/Components/NcLoadingIcon.js'

import {
	APP_API_PROXY_URL_PREFIX,
	APP_API_ROUTER_BASE,
	EX_APP_ID,
	EX_APP_MENU_ENTRY_NAME,
} from '../constants/AppAPI.js'

import { getFilePickerBuilder } from '@nextcloud/dialogs'
import '@nextcloud/dialogs/style.css'

export default {
	name: 'IframeView',
	components: {
		NcEmptyContent,
		AlertCircleIcon,
		NcLoadingIcon,
	},
	data() {
		const baseUrl = generateUrl(`${APP_API_PROXY_URL_PREFIX}/${EX_APP_ID}/`, {}, { noRewrite: true })
		const iframeSrcUrl = process.env.HARP_ENABLED ? baseUrl.replace('/index.php', '') : baseUrl

		return {
			error: null,
			loading: true,
			iframeBaseUrl: baseUrl,
			pageBaseUrl: generateUrl(`${APP_API_ROUTER_BASE}/${EX_APP_ID}/${EX_APP_MENU_ENTRY_NAME}`),
			iframeSrc: this.loadIframeSrcUrlRoute(baseUrl, iframeSrcUrl),
		}
	},
	mounted() {
		const timeout = setTimeout(() => {
			this.error = true
			this.loading = false
		}, 10000)
		this.$refs.iframe.addEventListener('load', (event) => {
			console.debug('iframe service loaded', event)
			this.loading = false
			clearTimeout(timeout)
		})
		this.$refs.iframe.addEventListener('error', (event) => {
			console.debug('iframe service error', event)
			this.error = true
			this.loading = false
		})

		window.addEventListener('message', (event) => {
			if (event.data.type === 'openNextcloudFilePicker') {
				this.showFilePicker(event)
			}
			if (event.data.type === 'routeChange') {
				this.handleIframeRouteChange(event.data.route)
			}
		})
	},
	methods: {
		showFilePicker(event) {
			if (!event.data.inputParamName) {
				return
			}
			const inputParamName = event.data.inputParamName
			getFilePickerBuilder(t('visionatrix', 'Select a file'))
				.setMultiSelect(false)
				.allowDirectories(false)
				.addMimeTypeFilter('image/*')
				.addButton({
					label: t('visionatrix', 'Select'),
					callback: (nodes) => {
						this.sendSelectedFilesToIframe(nodes, inputParamName)
					},
				})
				.build().pick().catch(() => {})
		},
		sendSelectedFilesToIframe(nodes, inputParamName) {
			const files = nodes.map((node) => {
				return {
					...node?._data,
				}
			})
			this.$refs.iframe.contentWindow.postMessage({ files, inputParamName }, '*')
		},
		loadIframeSrcUrlRoute(iframeSrcUrl) {
			const pageBaseUrl = generateUrl(`${APP_API_ROUTER_BASE}/${EX_APP_ID}/${EX_APP_MENU_ENTRY_NAME}`)
			const pagePathRoute = window.location.pathname
				.replace(pageBaseUrl, '')
			// fix duplicate slashes, remove starting slash
			const normalizedPath = pagePathRoute.replace(/^\//, '').replace(/\/\//g, '/')

			if (iframeSrcUrl.endsWith('/')) {
				iframeSrcUrl = `${iframeSrcUrl}${normalizedPath}`
			} else {
				iframeSrcUrl = `${iframeSrcUrl}/${normalizedPath}`
			}

			// always append ending slash if it is not present
			if (!iframeSrcUrl.endsWith('/')) {
				iframeSrcUrl += '/'
			}

			if (process.env.HARP_ENABLED) {
				iframeSrcUrl = iframeSrcUrl.replace('/index.php', '')
			}

			return iframeSrcUrl
		},
		handleIframeRouteChange(route) {
			if (!route) {
				return
			}

			const pageBaseUrl = generateUrl(`${APP_API_ROUTER_BASE}/${EX_APP_ID}/${EX_APP_MENU_ENTRY_NAME}`)
			// Update window route part after the root path
			let normalizedPath = route
			if (route !== '/') {
				 normalizedPath = route.replace(/^\//, '').replace(/\/\//g, '/')
				if (this.pageBaseUrl.endsWith('/')) {
					this.pageBaseUrl = `${pageBaseUrl}${normalizedPath}`
				} else {
					this.pageBaseUrl = `${pageBaseUrl}/${normalizedPath}`
				}
			} else {
				this.pageBaseUrl = pageBaseUrl
			}

			// replace potential duplicate slashes
			this.pageBaseUrl = this.pageBaseUrl.replace(/\/\//g, '/')

			// change the window path, without history
			window.history.replaceState({}, '', this.pageBaseUrl)
		},
	},
}
</script>

<style>
.iframe-ui-viewer {
	display: flex;
	justify-content: center;
	align-items: center;
	height: 100%;
}

.visionatrix__iframe {
	width: 100%;
	height: 100%;
	flex-grow: 1;
	background-color: #1c1b22;
}
</style>
