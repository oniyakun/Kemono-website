<template>
<div id="base-profile">
	<b-loading :active="!profile" :canCancel="true"></b-loading>
	<div class="profile-details" v-if="profile">
		<img :src="profile.avatar || require('@/../static/images/404.jpg')" class="avatar-profile">
		<p class="username">{{ profile.username }}
			<b-tag v-for="role of profile.roles" :key="role" type="is-info" :closable="canEditRoles" @close="revokeRole(role)">{{ role | deCamelCase }}</b-tag>
			<button class="button is-success is-small" v-if="canEditRoles" @click="promptGrantRole">Add role</button>
		</p>
		<div class="likes">
			<b-icon icon="thumb-up"></b-icon>
			<span>{{ profile.likesReceived | humanize }} {{$t('m.likebtn')}}</span>
		</div>
		<div class="favorites">
			<b-icon icon="heart"></b-icon>
			<span>{{ profile.favoritesReceived | humanize }} {{$t('m.favbtn')}}</span>
		</div>
		<div class="stats">
			<p>{{$t('m.jointime')}} <timeago :since="profile.createdAt"></timeago></p>
			<p>{{$t('m.posted')}} {{ profile.uploads | humanize }} {{$t('m.images')}}</p>
			<p>{{$t('m.hasgiven')}} {{ profile.likes.length | humanize }} {{$t('m.like')}} {{$t('m.and')}} {{ profile.favorites.length | humanize }} {{$t('m.fav')}}</p>
		</div>
	</div>
	<div class="post-grid-wrapper">
		<b-tabs class="post-tabs" type="is-boxed" position="is-centered" @change="changeTab">
			<b-tab-item :label="$t('m.profileposts')" icon="format-list-bulleted"></b-tab-item>
			<b-tab-item :label="$t('m.profilelikes')" icon="thumb-up"></b-tab-item>
			<b-tab-item :label="$t('m.profilefav')" icon="heart"></b-tab-item>
			<b-tab-item :label="$t('m.profilepending')" icon="checkbox-multiple-marked-outline" v-if="profile && profile.id === user.id || userCanApprove"></b-tab-item>
		</b-tabs>
		<div class="pagination-wrapper top">
			<b-pagination
				:total="hitEnd ? posts.length : posts.length + 1"
				:current="page"
				order="is-centered"
				:per-page="1"
				@change="changePage">
			</b-pagination>
		</div>
		<transition-group name="fade">
			<div class="page" v-for="(_page, i) of posts" :key="i" v-if="page === i + 1">
				<div class="columns is-multiline is-centered">
					<div class="column is-one-third" v-for="(post, i2) of _page" :key="i2">
						<post-card :post="post" :blur="!allowNSFW && post.nsfw" />
					</div>
				</div>
			</div>
		</transition-group>
		<div class="pagination-wrapper bottom">
			<b-pagination
				:total="hitEnd ? posts.length : posts.length + 1"
				:current="page"
				order="is-centered"
				:per-page="1"
				@change="changePage">
			</b-pagination>
		</div>
	</div>
</div>
</template>

<script>
import PostCard from '@/components/PostCard';

export default {
	data() {
		return {
			THUMBNAIL_BASE_URL,
			posts: [],
			page: 1,
			profile: null,
			mode: 'uploads',
			hitEnd: false
		};
	},
	components: {
		PostCard
	},
	computed: {
		user() {
			return this.$store.state.user;
		},
		canEditRoles() {
			return this.user && this.user.roles && this.user.roles.includes('admin')
		},
		userCanApprove() {
			return this.$store.getters.userCanApprove;
		},
		loggedIn() {
			return this.$store.state.loggedIn;
		},
		allowNSFW() {
			return this.$store.getters.NSFWImages;
		}
	},
	methods: {
		async changePage(page) {
			if (this.mode !== 'pending' && page > this.page) {
				if (this.posts.length !== 0 && this.posts.length % 3 === 0 && this.posts[this.posts.length - 1].length % 9 === 0 && this.posts.length <= page) {
					if (this.mode === 'uploads')
						await this.getUploads();
					else
						await this.getImages();
				} else if (page >= this.posts.length)
					this.hitEnd = true
			}
			this.page = page;
		},
		async getUser() {
			try {
				if (!this.$route.params.id)
					return;

				let response = await this.$http.get(`${API_BASE_URL}user/${this.$route.params.id}`, {
					headers: {
						'Authorization': localStorage.getItem('token')
					}
				});

				this.profile = response.data.user;
				return;
			} catch(error) {
				console.error(error);
				return this.$dialog.alert({
					type: 'is-danger',
					title: this.$t('m.errgetuserdata'),
					message: error ? error.response && error.response.data.message || error.message : 'Unknown Error',
					hasIcon: true
				});
			}
		},
		async getImages() {
			try {
				if (!this.profile)
					return;

				let ids = this.mode === 'likes' ? this.profile.likes : this.profile.favorites;
				ids = ids.reverse().slice(this.posts.length * 9, this.posts.length * 9 + 27);

				let response = await this.$http.post(API_BASE_URL + 'batch/images', { ids }, {
					headers: {
						'Authorization': localStorage.getItem('token')
					}
				});

				if (response.data.images.length === 0)
					this.hitEnd = true;
				else {
					while (response.data.images.length > 0)
						this.posts.push(response.data.images.splice(0, 9));

					if (this.posts[this.posts.length - 1].length !== 9)
						this.hitEnd = true;
				}

				return response;
			} catch(error) {
				console.error(error);
				return this.$dialog.alert({
					type: 'is-danger',
					title: this.$t('m.errgetimgdata'),
					message: error ? error.response && error.response.data.message || error.message : 'Unknown Error',
					hasIcon: true
				});
			}
		},
		async getUploads() {
			try {
				if (!this.profile)
					return;

				let response = await this.$http.post(API_BASE_URL + 'images/search', {
					sort: 'recent',
					limit: 27,
					skip: this.posts.length * 9,
					uploader: {
						id: this.profile.id
					}
				}, {
					headers: {
						'Authorization': localStorage.getItem('token')
					}
				});

				if (response.data.images.length === 0)
					this.hitEnd = true;
				else {
					while (response.data.images.length > 0)
						this.posts.push(response.data.images.splice(0, 9));

					if (this.posts[this.posts.length - 1].length !== 9)
						this.hitEnd = true;
				}

				return response;
			} catch(error) {
				console.error(error);
				return this.$dialog.alert({
					type: 'is-danger',
					title: this.$t('m.errgetuserupload'),
					message: error ? error.response && error.response.data.message || error.message : 'Unknown Error',
					hasIcon: true
				});
			}
		},
		async getPending() {
			try {
				if (!this.profile || (this.profile.id !== this.user.id && !this.userCanApprove))
					return;

				let response = await this.$http.get(API_BASE_URL + 'pending/list', {
					params: { user: this.profile.id },
					headers: { Authorization: localStorage.getItem('token') }
				});

				this.hitEnd = true;

				while (response.data.images.length > 0)
					this.posts.push(response.data.images.splice(0, 9));

				return response;
			} catch(error) {
				console.error(error);
				return this.$dialog.alert({
					type: 'is-danger',
					title: this.$t('m.errgetuserupload'),
					message: error ? error.response && error.response.data.message || error.message : 'Unknown Error',
					hasIcon: true
				});
			}
		},
		changeTab(position) {
			switch (position) {
				case 0:
					this.mode = 'uploads';
					break;
				case 1:
					this.mode = 'likes';
					break;
				case 2:
					this.mode = 'favorites';
					break;
				case 3:
					this.mode = 'pending';
					break;
			}

			this.page = 1;
			this.posts = [];
			this.hitEnd = false; // false unless pending

			if (this.mode === 'uploads')
				this.getUploads();
			else if (this.mode === 'pending')
				this.getPending();
			else
				this.getImages();
		},
		async revokeRole(role) {
			try {
				let resp = await this.$http.patch(`${API_BASE_URL}user/${this.profile.id}/roles`, {
					role,
					action: 'revoke'
				}, { headers: { 'Authorization': localStorage.getItem('token') } });

				this.$parent.$delete(this.profile.roles, this.profile.roles.indexOf(role));

				return this.$snackbar.open({
					type: 'is-success',
					message: this.$t('m.rolerevoked1') + role + this.$t('m.rolerevoked2') + this.profile.username,
					duration: 5000,
					position: 'is-bottom-right'
				});
			} catch (error) {
				console.error(error);
				return this.$dialog.alert({
					type: 'is-danger',
					title: this.$t('m.errrevokrole'),
					message: error ? error.response && error.response.data.message || error.message : 'Unknown Error',
					hasIcon: true
				});
			}

		},
		promptGrantRole() {
			return this.$dialog.prompt({
				title: 'Add role',
				message: this.$t('m.addrole') + this.profile.username + `?`,
				inputAttrs: { },
				confirmText: this.$t('m.addroleconfirm'),
				onConfirm: role => this.grantRole(role)
			});
		},
		async grantRole(role) {
			try {
				let resp = await this.$http.patch(`${API_BASE_URL}user/${this.profile.id}/roles`, {
					role,
					action: 'grant'
				}, { headers: { 'Authorization': localStorage.getItem('token') } });

				this.profile.roles.push(role);

				return this.$snackbar.open({
					type: 'is-success',
					message: `Role ${role} given to ${this.profile.username}`,
					duration: 5000,
					position: 'is-bottom-right'
				});
			} catch (error) {
				console.error(error);
				return this.$dialog.alert({
					type: 'is-danger',
					title: this.$t('m.errgrantrole'),
					message: error ? error.response && error.response.data.message || error.message : 'Unknown Error',
					hasIcon: true
				});
			}
		}
	},
	async mounted() {
		await this.getUser();
		if (this.mode === 'uploads')
			this.getUploads();
		else
			this.getImages();
	},
	watch: {
		async $route() {
			await this.getUser();

			this.page = 1;
			this.posts = [];
			this.hitEnd = false;

			if (this.mode === 'uploads')
				this.getUploads();
			else
				this.getImages();
		}
	}
}
</script>

<style lang="sass">
#base-profile
	.profile-details
		margin: 0 auto 40px
		max-width: 600px
		display: grid
		grid-template-columns: 50% 50%
		grid-template-rows: 256px 60px 40px 40px
		grid-gap: 5px 10px
		align-items: center
		justify-items: center
		.likes, .favorites
			grid-column: 1
			font-size: 24px
			.icon
				margin-right: 6px
		.likes
			grid-row: 3
			color: #209cee
		.favorites
			grid-row: 4
			color: #ff3860
		.avatar-profile
			grid-column: 1 / span 2
			grid-row: 1
			max-width: 256px
			max-height: 256px
			border-radius: 5px
		.username
			grid-column: 1 / span 2
			grid-row: 2
			font-family: 'Nunito', sans-serif
			font-size: 36px
			line-height: 60px
			.tag
				position: relative
				margin-left: 10px
				height: 27px
				border-radius: 2px
				top: -3px
				vertical-align: middle
			.button
				top: -3px
				vertical-align: middle
		.stats
			grid-column: 2
			grid-row: 3 / span 2
	.post-grid-wrapper
		.pagination-wrapper
			margin: auto
			max-width: 390px
			&.top
				margin: 16px auto
			&.bottom
				margin-top: 16px
		.page .columns .column
			margin: auto 0
	.fade-enter-active, .fade-leave-active
		transition: opacity .2s ease-in-out both
	.fade-enter, .fade-leave-to
		opacity: 0

	// IDK why this works, but if you remove it then changing page brings you back to the top
	.fade-enter-active + .page, .page + .fade-enter-active
		display: none

</style>
