<template>
	<div class="nextcloud-filepicker">
		<div id="trigger-buttons">
			<div v-if="enableGetFilesPath" @click="getFilesPath">
				<slot name="get-files-path">
					<button>
						<span class="icon icon-download" />
						{{ t('filepicker', 'Get files path') }}
					</button>
				</slot>
			</div>
			<div v-if="enableGetFilesLink" @click="onGetFilesLinkClick">
				<slot name="get-files-link">
					<button>
						<span class="icon icon-public" />
						{{ t('filepicker', 'Get files link') }}
					</button>
				</slot>
			</div>
			<div v-if="enableDownloadFiles" @click="downloadFiles">
				<slot name="download-files">
					<button>
						<span class="icon icon-download" />
						{{ t('filepicker', 'Download files') }}
					</button>
				</slot>
			</div>
			<div v-if="enableGetSaveFilePath" @click="getSaveFilePath">
				<slot name="get-save-file-path">
					<button>
						<span class="icon icon-upload" />
						{{ t('filepicker', 'Get save file path') }}
					</button>
				</slot>
			</div>
			<div v-if="enableGetUploadFileLink" @click="getUploadFileLink">
				<slot name="get-upload-fileLink">
					<button>
						<span class="icon icon-upload" />
						{{ t('filepicker', 'Get file upload link') }}
					</button>
				</slot>
			</div>
			<div v-if="enableUploadFiles" @click="openFileInput">
				<slot name="open-file-input">
					<button>
						<span class="icon icon-upload" />
						{{ t('filepicker', 'Upload files') }}
					</button>
				</slot>
			</div>
		</div>
		<input v-show="false"
			id="file-input"
			ref="myFiles"
			type="file"
			:multiple="multipleUpload"
			@change="onFileInputChange">
		<NcModal v-if="isOpen"
			:can-close="false"
			size="large"
			:style="cssVars"
			@close="close">
			<FilePicker
				:get-title="getTitle"
				:put-title="putTitle"
				:loading-directory="loadingDirectory"
				:uploading-files="uploadingFiles"
				:upload-progress="uploadProgress"
				:download-progress="downloadProgress"
				:downloading-files="downloadingFiles"
				:dark-mode="myDarkMode"
				:current-path="currentPath"
				:current-elements="currentElements"
				:connected="connected"
				:mode="mode"
				:multiple-download="multipleDownload"
				:quota="quota"
				:quota-loading="quotaLoading"
				:display-quota-refresh="displayQuotaRefresh"
				:files-to-upload="filesToUpload"
				@close="close(true)"
				@refresh-quota="updateWebdavQuota"
				@folder-clicked="onFolderClicked"
				@selection-changed="onSelectionChanged"
				@create-directory="createDirectory"
				@validate="onValidate"
				@breadcrumb-hash-changed="onBreadcrumbChange">
				<template #file-icon="{node}">
					<NextcloudFileIcon
						:nc-url="url"
						:node="node"
						:dark-mode="myDarkMode"
						:display-previews="displayPreviews"
						:client="client" />
				</template>
			</FilePicker>
		</NcModal>
	</div>
</template>

<script>
import { setLanguage, t, n } from '../translation.js'
import { WebDavFetchClient } from '../webdavFetchClient.js'
import { initVueAuthenticate } from '../services/auth.js'
import { colorOpacity } from '../utils.js'

import '../../css/filepicker.scss'

import axios from 'axios'
import moment from '@nextcloud/moment'

import FilePicker from './FilePicker.vue'
import NextcloudFileIcon from './NextcloudFileIcon.vue'

import NcModal from '@nextcloud/vue/dist/Components/NcModal.js'

export default {
	name: 'NcWebdavFilePicker',

	components: {
		NcModal,
		FilePicker,
		NextcloudFileIcon,
	},

	props: {
		/* === reactive props === */
		// Nextcloud base URL
		ncUrl: {
			type: String,
			required: true,
		},
		// Nextcloud user name
		ncLogin: {
			type: String,
			default: '',
		},
		// Nextcloud user password/app password/OAuth access token
		ncPassword: {
			type: String,
			default: '',
		},
		// OAuth access token if you absolutely want to use Bearer Authorization header (if not, using a token as a password works fine)
		ncAccessToken: {
			type: String,
			default: '',
		},
		// OIDC configuration
		ncOidcConfig: {
			type: Object,
			default: () => null,
		},
		// URL to get the OIDC configuration
		oidcConfigLocation: {
			type: String,
			default: '',
		},
		// Include cookies in WebDav and OCS requests if this is true
		useCookies: {
			type: Boolean,
			default: false,
		},
		// use WebAppPassword login flow
		useWebapppassword: {
			type: Boolean,
			default: true,
		},
		/* === props to control the fp component from the parent one === */
		// file picker mode to determine what is done when the picker is opened
		pickerMode: {
			type: String,
			default: '',
		},
		// prop to open the file picker if you don't want to use the buttons
		pickerIsOpen: {
			type: Boolean,
			default: false,
		},
		/* === options === */
		// enable multiple selection in all download modes
		multipleDownload: {
			type: Boolean,
			default: true,
		},
		// enable multiple local files selection when uploading
		multipleUpload: {
			type: Boolean,
			default: true,
		},
		// close the picker on network error for getFilesLink, uploadFiles and downloadFiles (like when password policy refuses a link password)
		closeOnError: {
			type: Boolean,
			default: false,
		},
		// file picker title
		getTitle: {
			type: String,
			default: null,
		},
		putTitle: {
			type: String,
			default: null,
		},
		// theming (reactive too)
		themeColor: {
			type: String,
			default: '#0082c9',
			validator: (value) => {
				return value.match(/^#[0-9a-fA-F]{6}$/)
			},
		},
		darkMode: {
			type: Boolean,
			default: false,
		},
		displayPreviews: {
			type: Boolean,
			default: true,
		},
		displayQuotaRefresh: {
			type: Boolean,
			default: false,
		},
		/* === toggle buttons === */
		// display the button to get files path
		enableGetFilesPath: {
			type: Boolean,
			default: false,
		},
		// display the button to get files links
		enableGetFilesLink: {
			type: Boolean,
			default: false,
		},
		// display the button to download files
		enableDownloadFiles: {
			type: Boolean,
			default: false,
		},
		// display the button to get a save file path
		enableGetSaveFilePath: {
			type: Boolean,
			default: false,
		},
		// display the button to get webdav upload link
		enableGetUploadFileLink: {
			type: Boolean,
			default: false,
		},
		// display the button to upload local files
		enableUploadFiles: {
			type: Boolean,
			default: false,
		},
		// optional language for translation (xx-XX and xx locales accepted), use browser locale if null
		language: {
			type: String,
			default: null,
		},
	},

	data() {
		return {
			t,
			n,
			// initialize values with props
			login: this.ncLogin,
			password: this.ncPassword,
			accessToken: this.ncAccessToken,
			url: this.ncUrl,
			mainColor: this.themeColor || '#0082c9',
			myDarkMode: this.darkMode,
			// state data
			client: null,
			connected: false,
			isOpen: false,
			currentElements: [],
			currentElementsByPath: {},
			currentPath: '/',
			selection: [],
			quota: null,
			quotaLoading: false,
			loadingDirectory: false,
			uploadingFiles: false,
			uploadProgress: 0,
			downloadingFiles: false,
			downloadProgress: 0,
			// link settings
			linkLabel: '',
			// modes : getFilesPath, downloadFiles, getFilesLink, getSaveFilePath, uploadFiles, getUploadFileLink
			mode: '',
			loginWindow: null,
			filesToUpload: [],
			// translations
			gt: null,
			// OIDC
			oidcConfig: null,
			oidcAuthInstance: null,
		}
	},

	computed: {
		cssVars() {
			return {
				'--color-primary': this.mainColor,
				'--color-primary-element': this.mainColor,
				'--color-primary-text': '#ffffff',
				'--color-primary-element-light': this.mainColorLight,
				'--color-primary-light': this.mainColorLighter,
				'--color-main-background': this.mainBackgroundColor,
				'--color-border': this.colorBorder,
				'--color-border-dark': this.colorBorderDark,
				'--color-main-text': this.mainTextColor,
				'--color-text-lighter': this.colorTextLighter,
				'--color-background-hover': this.colorBackgroundHover,
				'--color-background-dark': this.colorBackgroundDark,
				'--color-background-darker': this.colorBackgroundDarker,
				'--border-radius-large': '10px',
				'--default-font-size': '15px',
				'--font-face': '-apple-system,BlinkMacSystemFont,\'Segoe UI\',Roboto,Oxygen-Sans,Cantarell,Ubuntu,\'Helvetica Neue\',Arial,\'Noto Color Emoji\',sans-serif,\'Apple Color Emoji\',\'Segoe UI Emoji\',\'Segoe UI Symbol\'',
			}
		},
		mainTextColor() {
			return this.myDarkMode
				? '#d8d8d8'
				: '#222'
		},
		mainBackgroundColor() {
			return this.myDarkMode
				? '#131313'
				: 'white'
		},
		colorBorder() {
			return this.myDarkMode
				? '#2a2a2a'
				: '#ededed'
		},
		colorBorderDark() {
			return this.myDarkMode
				? '#3c3c3c'
				: '#dbdbdb'
		},
		colorTextLighter() {
			return this.myDarkMode
				? '#b4b4b4'
				: '#767676'
		},
		colorBackgroundDark() {
			return this.myDarkMode
				? '#222222'
				: '#ededed'
		},
		colorBackgroundDarker() {
			return this.myDarkMode
				? '#2c2c2c'
				: '#dbdbdb'
		},
		colorBackgroundHover() {
			return this.myDarkMode
				? '#0a0a0a'
				: '#f5f5f5'
		},
		mainColorLight() {
			return this.myDarkMode
				? colorOpacity(this.mainColor, 0.6)
				: colorOpacity(this.mainColor, 0.4)
		},
		mainColorLighter() {
			return this.myDarkMode
				? colorOpacity(this.mainColor, 0.8)
				: colorOpacity(this.mainColor, 0.2)
		},
		authUrl() {
			return this.url + '/index.php/apps/webapppassword'
		},
		davUrl() {
			return this.url + '/remote.php/dav/files'
		},
	},

	watch: {
		ncLogin() {
			this.updateLogin(this.ncLogin)
		},
		ncPassword() {
			this.updatePassword(this.ncPassword)
		},
		ncUrl() {
			this.updateUrl(this.ncUrl)
		},
		ncAccessToken() {
			this.updateAccessToken(this.ncAccessToken)
		},
		themeColor() {
			this.setMainColor(this.themeColor)
		},
		darkMode() {
			this.setDarkMode(this.darkMode)
		},
		// let parent component control the fp
		pickerMode() {
			this.mode = this.pickerMode
		},
		pickerIsOpen() {
			if (this.pickerIsOpen) {
				this.isOpen = true
				this.getFolderContent()
			}
		},
	},

	beforeMount() {
		if (this.language) {
			setLanguage(this.language)
		}
		if (this.ncOidcConfig) {
			this.oidcConfig = this.ncOidcConfig
			this.initOidcAuthentication()
		} else if (this.oidcConfigLocation) {
			this.getOidcConfig()
		}
	},

	methods: {
		resetFilePicker() {
			this.client = null
			this.connected = false
			this.quota = null
			this.uploadingFiles = false
		},
		// OIDC stuff
		async getOidcConfig() {
			const config = await fetch(this.oidcConfigLocation)
			this.oidcConfig = await config.json()
			this.initOidcAuthentication()
		},
		initOidcAuthentication() {
			this.oidcAuthInstance = initVueAuthenticate(this.oidcConfig)
			this.oidcAuthInstance.mgr.events.addAccessTokenExpiring(() => {
				console.debug('auth token is expiring')
			})
			this.oidcAuthInstance.mgr.events.addAccessTokenExpired(() => {
				console.debug('auth token has expired')
			})
			// create a webdav client after a successful login/renew
			this.oidcAuthInstance.mgr.events.addUserLoaded(() => {
				console.debug('token obtained or renewed -> create or recreate the client')
				const clientWasNull = this.client === null
				this.createClientFromOidcAuth()
				// only load the content if we just created the client, otherwise it's most likely a token renew
				// and we want it to be transparent
				if (clientWasNull) {
					this.getFolderContent()
				}
			})
		},
		createClientFromOidcAuth() {
			const oidcToken = this.oidcAuthInstance.getToken()
			const userObject = this.oidcAuthInstance.getStoredUserObject()
			console.debug('createClientFromOidcAuth, token expires in', userObject.expires_in)
			const userId = userObject?.profile?.preferred_username
			this.client = new WebDavFetchClient({
				url: this.davUrl + '/' + userId,
				userId,
				token: {
					access_token: oidcToken,
					token_type: 'Bearer',
				},
				useCookies: this.useCookies,
			})
		},
		updateUrl(newValue) {
			this.resetFilePicker()
			this.url = newValue
		},
		updateLogin(newValue) {
			this.resetFilePicker()
			this.login = newValue
		},
		updatePassword(newValue) {
			this.resetFilePicker()
			this.password = newValue
		},
		updateAccessToken(newValue) {
			this.resetFilePicker()
			this.accessToken = newValue
		},
		setMainColor(color) {
			this.mainColor = color
		},
		setDarkMode(isEnabled) {
			this.myDarkMode = isEnabled
		},
		createClient() {
			// reset
			this.currentElements = []
			this.currentPath = '/'

			// basic http auth (classic password, app password or even oauth token)
			if (this.login && this.password) {
				this.client = new WebDavFetchClient({
					url: this.davUrl + '/' + this.login,
					userId: this.login,
					username: this.login,
					password: this.password,
					useCookies: this.useCookies,
				})
				this.getFolderContent()
			} else if (this.login && this.accessToken) {
				// OAuth2 token
				this.client = new WebDavFetchClient({
					url: this.davUrl + '/' + this.login,
					userId: this.login,
					token: {
						access_token: this.accessToken,
						token_type: 'Bearer',
					},
					useCookies: this.useCookies,
				})
				this.getFolderContent()
			} else if (this.oidcConfig) {
				// OIDC client
				if (this.oidcAuthInstance.isAuthenticated()) {
					this.createClientFromOidcAuth()
					this.getFolderContent()
				} else {
					this.oidcAuthInstance.authenticate()
				}
			} else if (this.login) {
				// no auth, no web login
				this.client = new WebDavFetchClient({
					url: this.davUrl + '/' + this.login,
					userId: this.login,
					username: this.login,
					useCookies: this.useCookies,
				})
				this.getFolderContent()
			} else if (this.useWebapppassword) {
				// web login flow
				const authUrl = this.authUrl + '?target-origin=' + encodeURIComponent(window.location.href)
				this.loginWindow = window.open(
					authUrl,
					'Nextcloud Login',
					'width=400,height=400,menubar=no,scrollbars=no,status=no,titlebar=no,toolbar=no'
				)
				window.addEventListener('message', this.onReceiveWindowMessage)
			}
		},
		onReceiveWindowMessage(e) {
			const data = e.data
			if (data.type === 'webapppassword') {
				if (this.loginWindow !== null) {
					this.loginWindow.close()
				}
				window.removeEventListener('message', this.onReceiveWindowMessage)
				this.login = data.loginName
				this.password = data.token
				this.createClient()
			}
		},
		openFileInput() {
			this.$refs.myFiles.click()
		},
		getFilesPath() {
			this.mode = 'getFilesPath'
			this.openFilePicker()
		},
		onGetFilesLinkClick(e) {
			this.getFilesLink()
		},
		getFilesLink(options = {}) {
			this.mode = 'getFilesLink'
			this.linkLabel = options.linkLabel || ''
			this.openFilePicker()
		},
		uploadFiles() {
			this.mode = 'uploadFiles'
			this.openFilePicker()
		},
		downloadFiles() {
			this.mode = 'downloadFiles'
			this.openFilePicker()
		},
		getSaveFilePath() {
			this.mode = 'getSaveFilePath'
			this.openFilePicker()
		},
		getUploadFileLink() {
			this.mode = 'getUploadFileLink'
			this.openFilePicker()
		},
		openFilePicker() {
			this.isOpen = true
			this.getFolderContent()
		},
		onUnauthorized(response) {
			const detail = { response }
			this.$emit('filepicker-unauthorized', detail)
			const event = new CustomEvent('filepicker-unauthorized', { detail })
			document.dispatchEvent(event)
			// this.close()
		},
		async getFolderContent(path = null) {
			if (path) {
				this.currentPath = path
			}
			if (this.client === null) {
				this.createClient()
			} else {
				this.selection = []
				this.currentElementsByPath = {}
				this.loadingDirectory = true
				try {
					const getQuotaFromPropfind = (path === null || path === '/' || path === '')
					const directoryContent = await this.client.getDirectoryContents(this.currentPath, getQuotaFromPropfind)
					console.debug('webdav client result', directoryContent)
					if (getQuotaFromPropfind && directoryContent.quota) {
						this.quota = directoryContent.quota
					}
					this.currentElements = directoryContent.nodes.map((el) => {
						this.currentElementsByPath[el.filename] = el
						return {
							...el,
							lastmod_ts: moment(el.lastmod).unix(),
						}
					})
					console.debug(this.currentElements)
					this.connected = true
				} catch (error) {
					console.error(error)
					if (error.response?.status === 401) {
						this.onUnauthorized(error.response)
					}
					this.resetFilePicker()
				}
				this.loadingDirectory = false
			}
		},
		close(manually = false) {
			this.isOpen = false
			this.$emit('closed')
			const closedEvent = new CustomEvent('filepicker-closed')
			document.dispatchEvent(closedEvent)
			if (manually) {
				this.$emit('manually-closed')
				const manuallyClosedEvent = new CustomEvent('filepicker-manually-closed')
				document.dispatchEvent(manuallyClosedEvent)
			}
		},
		onFolderClicked(path) {
			this.getFolderContent(path)
		},
		onSelectionChanged(selection) {
			this.selection = selection
		},
		onBreadcrumbChange(path) {
			this.getFolderContent(path)
		},
		async onValidate(options = {}) {
			if (this.mode === 'uploadFiles') {
				console.debug('upload to ' + this.currentPath)
				console.debug(this.filesToUpload)
				this.webdavUploadFiles()
			} else if (this.mode === 'getFilesPath') {
				console.debug('get file path in ' + this.currentPath)
				const detail = { selection: this.selection }
				// for parent component
				this.$emit('get-files-path', detail)
				// for potential global listener
				const event = new CustomEvent('get-files-path', { detail })
				document.dispatchEvent(event)
				this.close()
			} else if (this.mode === 'getFilesLink') {
				const shareLinksResult = await this.getFilesShareLink(this.selection, options)
				const shouldClosePicker = shareLinksResult.error === false || this.closeOnError
				const ocsUrl = this.url + '/ocs/v2.php/apps/files_sharing/api/v1/shares'
				const genericShareLink = this.url + '/index.php/s/TOKEN'
				const detail = {
					pathList: this.selection,
					ocsUrl,
					genericShareLink,
					shareLinks: shareLinksResult.publicLinks,
					linkOptions: options,
					error: shareLinksResult.error,
					pickerStillOpen: !shouldClosePicker,
				}
				// for parent component
				this.$emit('get-files-link', detail)
				// for potential global listener
				const event = new CustomEvent('get-files-link', { detail })
				document.dispatchEvent(event)
				if (shouldClosePicker) {
					this.close()
				}
			} else if (this.mode === 'getSaveFilePath') {
				console.debug('user wants to save in ' + this.currentPath)
				const detail = { path: this.currentPath }
				// for parent component
				this.$emit('get-save-file-path', detail)
				// for potential global listener
				const event = new CustomEvent('get-save-file-path', { detail })
				document.dispatchEvent(event)
				this.close()
			} else if (this.mode === 'getUploadFileLink') {
				console.debug('user wants to get an upload link in ' + this.currentPath)
				if (!this.password && this.accessToken) {
					console.error('Upload links can\'t be generated when using OAuth, you can provide the OAuth token as a normal password.')
					this.close()
					return
				}
				const uploadPath = this.currentPath.replace(/\/$/, '') + '/file.txt'
				try {
					const uploadLink = this.client.getFileUploadLink(uploadPath)
					const detail = {
						link: uploadLink,
						targetDir: this.currentPath,
					}
					// for parent component
					this.$emit('upload-path-link-generated', detail)
					// for potential global listener
					const event = new CustomEvent('upload-path-link-generated', { detail })
					document.dispatchEvent(event)
				} catch (error) {
					console.error('Impossible to generate upload link')
				}
				this.close()
			} else if (this.mode === 'downloadFiles') {
				console.debug('user wants to download files')
				console.debug(this.selection)
				this.webdavDownload()
			}
		},
		async getFilesShareLink(pathList, options) {
			// create shared access with OCS API
			// problem : CORS headers don't allow this for the moment,
			// this could be done by adding a global origin whitelist in NC server
			const url = this.url + '/ocs/v2.php/apps/files_sharing/api/v1/shares'
			const shareLinksResult = {
				error: false,
				publicLinks: [],
			}
			let path
			for (let i = 0; i < pathList.length; i++) {
				path = pathList[i]
				const publicLink = {
					path,
				}
				try {
					const headers = new Headers()
					this.client.appendAuthHeader(headers)
					headers.append('OCS-APIRequest', 'true')
					headers.append('Content-Type', 'application/json')
					headers.append('Accept', 'application/json')
					const req = {
						path,
						shareType: 3,
						label: this.linkLabel || 'File picker link',
						expireDate: options.expirationDate ? moment(options.expirationDate).format('YYYY-MM-DD') : null,
						password: options.protectionPassword ? options.protectionPassword : undefined,
					}

					const rawResponse = await fetch(url, {
						method: 'POST',
						credentials: this.client.credentialsMode,
						headers,
						body: JSON.stringify(req),
					})
					const response = await rawResponse.json()
					const creationSuccess = rawResponse.status < 400
					if (creationSuccess) {
						publicLink.url = response.ocs.data.url
					} else {
						publicLink.error = response?.ocs?.meta?.message ?? ''
						shareLinksResult.error = true
					}

					if (creationSuccess && options.allowEdition) {
						const shareId = response.ocs.data.id
						const putUrl = url + '/' + shareId
						const putReq = {
							permissions: options.allowEdition ? 3 : undefined,
						}
						const useFetch = true
						if (useFetch) {
							const putHeaders = new Headers()
							this.client.appendAuthHeader(putHeaders)
							putHeaders.append('OCS-APIRequest', 'true')
							putHeaders.append('Content-Type', 'application/json')
							putHeaders.append('Accept', 'application/json')
							const rawEditionResponse = await fetch(putUrl, {
								method: 'PUT',
								credentials: this.client.credentialsMode,
								headers: putHeaders,
								body: JSON.stringify(putReq),
							})
							const editionSuccess = rawEditionResponse.status < 400
							if (!editionSuccess) {
								console.debug('Impossible to edit shared access')
								const editionResponse = await rawEditionResponse.json()
								publicLink.editionError = editionResponse?.ocs?.meta?.message ?? ''
							}
						} else {
							try {
								await axios.put(putUrl, putReq, {
									headers: {
										...this.client.getAuthHeader(),
										'OCS-APIRequest': 'true',
										'Content-Type': 'application/json',
									},
									// this has no effect, cookies are passed anyway
									withCredentials: false,
								})
							} catch (error) {
								console.error('Impossible to edit shared access')
								console.error(error.response?.data?.ocs?.meta?.message)
								console.error(error)
								shareLinksResult.error = error
								return shareLinksResult
							}
						}
					}
				} catch (error) {
					console.error('Impossible to create public links')
					console.error(error)
					shareLinksResult.error = error
					return shareLinksResult
				}
				shareLinksResult.publicLinks.push(publicLink)
			}
			return shareLinksResult
		},
		async updateWebdavQuota() {
			if (this.client) {
				this.quotaLoading = true
				try {
					this.quota = await this.client.getQuota()
					console.debug('quota')
					console.debug(this.quota)
				} catch (error) {
					console.error(error)
					if (error.response?.status === 401) {
						this.onUnauthorized(error.response)
					}
					// do not reset the filepicker here as the problem might only be on the quota requests
					// this.resetFilePicker()
				}
				this.quotaLoading = false
			}
		},
		async webdavUploadFiles() {
			this.uploadingFiles = true
			this.uploadProgress = 0
			// calc total upload size
			let totalSize = 0
			let totalUploaded = 0
			for (let i = 0; i < this.filesToUpload.length; i++) {
				const file = this.filesToUpload[i]
				totalSize += file.size
			}

			const successFiles = []
			const errorFiles = []
			for (let i = 0; i < this.filesToUpload.length; i++) {
				const file = this.filesToUpload[i]
				try {
					await this.client.putFileContents(this.currentPath.replace(/\/$/, '') + '/' + file.name, file, {
						overwrite: true,
						onUploadProgress: progress => {
							// console.debug(`Uploaded ${progress.loaded} bytes of ${progress.total}`)
							console.debug(`uploaded ${totalUploaded + progress.loaded} on ${totalSize}`)
							this.uploadProgress = parseInt((totalUploaded + progress.loaded) / totalSize * 100)
						},
					})
				} catch (error) {
					// already exists or no permission
					if (error.response?.status === 412) {
						console.error('the file ' + file.name + ' could not be uploaded')
						console.error(error)
						errorFiles.push(file)
						continue
					} else {
						console.error(error)
						this.resetFilePicker()
						return
					}
				}
				successFiles.push(file)
				console.debug('UPLOAD success ' + file.name)
				totalUploaded += file.size
				this.uploadProgress = parseInt(totalUploaded / totalSize * 100)
			}

			const shouldClosePicker = errorFiles.length === 0 || this.closeOnError
			const detail = {
				targetDir: this.currentPath,
				successFiles,
				errorFiles,
				pickerStillOpen: !shouldClosePicker,
			}
			// for parent component
			this.$emit('files-uploaded', detail)
			// for potential global listener
			const event = new CustomEvent('files-uploaded', { detail })
			document.dispatchEvent(event)

			this.uploadingFiles = false
			this.uploadProgress = 0
			this.filesToUpload = []
			if (shouldClosePicker) {
				console.debug('close after upload')
				this.close()
			} else {
				this.getFolderContent()
				this.updateWebdavQuota()
			}
		},
		async webdavDownload() {
			this.downloadingFiles = true
			this.downloadProgress = 0
			const results = []
			const errorFilePaths = []
			let totalSize = 0
			let totalDownloaded = 0
			for (let i = 0; i < this.selection.length; i++) {
				const filepath = this.selection[i]
				const selectedElement = this.currentElementsByPath[filepath]
				totalSize += selectedElement.size
			}
			for (let i = 0; i < this.selection.length; i++) {
				const filepath = this.selection[i]
				const selectedElement = this.currentElementsByPath[filepath]
				try {
					console.debug('DOWNLOADING ' + filepath)
					const response = await this.client.getFileContents(filepath)
					if (response.status >= 400) {
						console.error('download error response:')
						console.debug(response)
						if (response.status === 401) {
							this.onUnauthorized(response)
							this.resetFilePicker()
							return
						}
						errorFilePaths.push(filepath)
						continue
					}
					const reader = response.body.getReader()

					// get total length
					// const contentLength = +response.headers.get('Content-Length')

					// read the data
					const chunks = [] // array of received binary chunks (comprises the body)
					while (true) {
						const { done, value } = await reader.read()
						if (done) {
							break
						}
						chunks.push(value)
						totalDownloaded += value.length
						this.downloadProgress = parseInt(totalDownloaded / totalSize * 100)
						console.debug(`Total progress ${this.downloadProgress} %`)
					}
					const file = new File(chunks, selectedElement.basename, { type: selectedElement.mime })
					results.push(file)
				} catch (error) {
					console.error('download error!')
					console.debug(error)
					if (error.response?.status === 401) {
						this.onUnauthorized(error.response)
						this.resetFilePicker()
						return
					}
					errorFilePaths.push(filepath)
					continue
				}
			}
			const shouldClosePicker = errorFilePaths.length === 0 || this.closeOnError
			const detail = {
				successFiles: results,
				errorFilePaths,
				pickerStillOpen: !shouldClosePicker,
			}
			// for parent component
			this.$emit('files-downloaded', detail)
			// for potential global listener
			const event = new CustomEvent('files-downloaded', { detail })
			document.dispatchEvent(event)
			if (shouldClosePicker) {
				this.close()
			}
			this.downloadingFiles = false
			this.downloadProgress = 0
		},
		async createDirectory(newDirectoryName) {
			const newDirectoryPath = this.currentPath.replace(/^\/$/, '') + '/' + newDirectoryName
			await this.webdavCreateDirectory(newDirectoryPath)
			this.getFolderContent(newDirectoryPath)
		},
		async webdavCreateDirectory(path) {
			try {
				await this.client.createDirectory(path)
			} catch (error) {
				console.error(error)
				if (error.response?.status === 401) {
					this.onUnauthorized(error.response)
				}
				this.resetFilePicker()
			}
		},
		onFileInputChange(e) {
			this.filesToUpload = [...this.$refs.myFiles.files]
			this.uploadFiles()
		},
	},
}
</script>

<style scoped lang="scss">
.nextcloud-filepicker {
	#trigger-buttons {
		display: flex;

		button {
			padding: 10px;
			font-weight: bold;
			border-radius: 100px;
			border: 1px solid lightgrey;
			cursor: pointer;
		}
	}

	.icon {
		min-width: 16px;
		min-height: 16px;
		display: inline-block;
	}
	.icon-upload {
		background-image: url('./../../img/upload.svg');
	}
	.icon-download {
		background-image: url('./../../img/download.svg');
	}
	.icon-public {
		background-image: url('./../../img/public.svg');
	}
}

::v-deep .modal-container {
	display: flex !important;
	min-height: 80%;
	border-radius: 10px !important;

	.icon {
		min-width: 16px;
		min-height: 16px;
		display: inline-block;
	}
	button {
		height: 44px;
	}
	button .icon {
		mask-size: 16px auto;
		mask-position: center;
		-webkit-mask-size: 16px auto;
		-webkit-mask-position: center;
		background-color: var(--color-main-text);
		margin-bottom: -3px;
		width: 20px;
	}
	.icon-close {
		mask: url('./../../img/close.svg') no-repeat;
		-webkit-mask: url('./../../img/close.svg') no-repeat;
	}
	.icon-add {
		mask: url('./../../img/add.svg') no-repeat;
		-webkit-mask: url('./../../img/add.svg') no-repeat;
	}
	.icon-history {
		mask: url('./../../img/history.svg') no-repeat;
		-webkit-mask: url('./../../img/history.svg') no-repeat;
	}
	.icon-checkmark {
		mask: url('./../../img/checkmark.svg') no-repeat;
		-webkit-mask: url('./../../img/checkmark.svg') no-repeat;
	}
	.icon-rename {
		mask: url('./../../img/rename.svg') no-repeat;
		-webkit-mask: url('./../../img/rename.svg') no-repeat;
	}
	.icon-password {
		mask: url('./../../img/password.svg') no-repeat;
		-webkit-mask: url('./../../img/password.svg') no-repeat;
	}
	.icon-public {
		mask: url('./../../img/public.svg') no-repeat;
		-webkit-mask: url('./../../img/public.svg') no-repeat;
	}
	.icon-disabled-user {
		mask: url('./../../img/disabled-user.svg') no-repeat;
		mask-size: 30px auto;
		mask-position: center;
		-webkit-mask: url('./../../img/disabled-user.svg') no-repeat;
		-webkit-mask-size: 30px auto;
		-webkit-mask-position: center;
		background-color: grey;
	}

	.empty-content {
		flex-grow: 1;
		color: lightgrey;

		.icon-disabled-user,
		.icon-folder {
			mask-size: 65px auto;
			-webkit-mask-size: 65px auto;
		}
	}

	input[type=text] {
		-moz-appearance: textfield;
		-webkit-appearance: textfield;
		background-color: var(--color-main-background);
		color: var(--color-main-text);
		border: 1px solid lightgrey;
		border-radius: 3px;
		padding: 0px 6px;
		height: 34px;
	}
}
</style>
