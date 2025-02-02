<template>
    <div class="topic">
        <TopicHeader
            v-if="!topicCreatorRemoved"
            :imageUrl="topic.imageUrl"
            :name="topic.name"
            :description="topic.description"
            :createdAt="topic.dateCreation"
            :createdBy="topic.User.username"
            :topicId="topic.id"
            :userId="currentUser.id"
            :followed="followed"
            :followers="topic.numberOfFollowers"
            @topic-followed="refreshFollowers"
            @topic-unfollowed="refreshFollowers"
        />
        <TopicHeader
            v-if="topicCreatorRemoved"
            :imageUrl="topic.imageUrl"
            :name="topic.name"
            :description="topic.description"
            :createdAt="topic.dateCreation"
            createdBy="un ancien utilisateur"
            :topicId="topic.id"
            :userId="currentUser.id"
            :followed="followed"
            :followers="topic.numberOfFollowers"
            @topic-followed="refreshFollowers"
            @topic-unfollowed="refreshFollowers"
        />

        <div class="topic__body row">
            <section class="topic__body__feed col-12 col-lg-9 pr-lg-5">
                <div class="topic__body__feed__publish pb-3 pr-lg-5 mb-2">
                    <b-form id="publishNewPostForm" @submit.stop.prevent="publishNewPost" novalidate>
                        <b-form-group id="newPostGroup" label="Exprimez-vous!" class="h3" label-for="newPostInput">
                            <b-form-textarea id="newPostInput" name="newPostInput" v-model="$v.form.newPost.$model" :state="validateState('newPost')" aria-describedby="newPostInputFeedback" type="text" rows="3" placeholder="Publiez quelque chose..." required></b-form-textarea>
                            <b-form-invalid-feedback id="newPostInputFeedback" :state="validateState('newPost')">Écrivez du contenu pour pouvoir le publier</b-form-invalid-feedback>
                        </b-form-group>

                        <div class="topic__body__feed__publish__submit">
                            <button type="submit" class="topic__body__feed__publish__submit__btn btn px-3 py-1">
                                Publier
                            </button>
                        </div>
                    </b-form>
                </div>

                <div class="topic__body__feed__posts row">
                    <p class="topic__body__feed__posts__empty col text-center h5 mt-5" v-if="noPosts">
                        Soyez le premier à publier quelque chose à propos de ce sujet!
                    </p>
                    
                    <Post 
                        v-for="post in posts"
                        :key="post.id"
                        :id="post.id"
                        :authorId="post.UserId"
                        :imageUrl="post.User.profilePicture"
                        :firstName="post.User.firstName"
                        :lastName="post.User.lastName"
                        :username="post.User.username"
                        :topicId="post.TopicId"
                        :content="post.content"
                        :numberOfLikes="post.likes"
                        :numberOfDislikes="post.dislikes"
                        :hasLiked="post.hasLiked"
                        :hasDisliked="post.hasDisliked"
                        :numberOfComments="post.numberOfComments"
                        @post-deleted="refreshPosts"
                        @post-updated="refreshPosts"
                        @post-liked="refreshPosts"
                        @post-disliked="refreshPosts"
                    />
                </div>
            </section>

            <aside class="topic__body__followers col-3 d-none d-lg-block">
                <TopicFollowers 
                    :hasFollowed="topic.hasFollowed"
                    :numberOfFollowers="topic.numberOfFollowers"
                    :followers="followers"
                />
            </aside>
        </div>
    </div>
</template>

<script>
import axios from 'axios'
import TopicHeader from '../components/TopicHeader'
import Post from '../components/Post'
import { mapGetters } from 'vuex'
import TopicFollowers from '../components/TopicFollowers'
import { validationMixin } from 'vuelidate'
import { required } from 'vuelidate/lib/validators'

export default {
    name: 'Topic',
    mixins: [validationMixin],
    components: {
        TopicHeader,
        Post,
        TopicFollowers
    },
    data() {
        return {
            topic: '',
            posts: [],
            noPosts: false,
            followed: false,
            followers: [],
            form: {
                newPost: ''
            },
            topicCreatorRemoved: false
        }
    },
    validations: {
        form: {
            newPost: {
                required
            }
        }
    },
    computed: {
        ...mapGetters(['currentUser', 'loggedIn']),
    },
    methods: {
        validateState(field) {
            const { $dirty, $error } = this.$v.form[field];
            return $dirty ? !$error : null;
        },
        async publishNewPost() {
            this.$v.form.$touch();
            if (this.$v.form.$anyError) {
                return;
            }
            const token = localStorage.getItem('token');
            const url = window.location.search;
            const searchUrl = new URLSearchParams(url);
            const topicId = searchUrl.get('id');
            const reqUrlPost = '/topics/' + topicId + '/posts';
            const createPost = await axios.post(reqUrlPost, {
                userId: this.currentUser.id,
                content: this.form.newPost
            }, {
                headers: {
                'Authorization': 'Bearer ' + token
                }
            });
            console.log(createPost.data);
            const reqUrlGet = reqUrlPost + '/?order=recent' 
            const postsRefreshed = await axios.get(reqUrlGet, {
                headers: {
                'Authorization': 'Bearer ' + token
                }
            });
            this.posts = postsRefreshed.data;
            this.form.newPost = '';
            if (this.noPosts) {
                this.noPosts = false;
            }
        },
        async refreshPosts() {
            const token = localStorage.getItem('token');
            const url = window.location.search;
            const searchUrl = new URLSearchParams(url);
            const topicId = searchUrl.get('id');
            this.posts = [];
            const reqUrl = '/topics/' + topicId + '/posts/?order=recent';
            const postsRefreshed = await axios.get(reqUrl, {
                headers: {
                'Authorization': 'Bearer ' + token
                }
            });
            if (postsRefreshed.data.length == 0) {
                this.noPosts = true;
            } else {
                this.posts = postsRefreshed.data;
            }
        }, 
        async refreshFollowers() {
            const token = localStorage.getItem('token');
            const url = window.location.search;
            const searchUrl = new URLSearchParams(url);
            const topicId = searchUrl.get('id');
            const reqUrlInfo = '/topics/' + topicId;
            const topicInfo = await axios.get(reqUrlInfo, {
                headers: {
                'Authorization': 'Bearer ' + token
                }
            });
            if (topicInfo.data.User == null) {
                this.topicCreatorRemoved = true;
            }
            this.topic = topicInfo.data;
            this.topic.hasFollowed = Array.from(this.topic.hasFollowed).filter(char => char !== ',');
            if (this.topic.hasFollowed.includes(this.currentUser.id.toString())) {
                this.followed = true;
            } else {
                this.followed = false;
            }
            const config = {
                headers: {
                    'Authorization': 'Bearer ' + token
                },
                params: {
                    followers: this.topic.hasFollowed,
                }
            };
            const topicFollowers = await axios.get('/users/?limit=10', config);
            this.followers = topicFollowers.data;
        }
    },
    async beforeMount() {
        if (!localStorage.getItem('token')) {
            this.$store.dispatch('changeLoginState', false);
            this.$router.push('/');
        } else {
            this.$store.dispatch('changeLoginState', true);
            this.$store.dispatch('storeUserId', parseInt(localStorage.getItem('userId')));
            const isAdmin = (localStorage.getItem('admin') === 'true') ? true : false;
            this.$store.dispatch('storeIsAdmin', isAdmin);
            const token = localStorage.getItem('token');
            const url = window.location.search;
            const searchUrl = new URLSearchParams(url);
            const topicId = searchUrl.get('id');
            const reqUrlInfo = '/topics/' + topicId;
            const topicInfo = await axios.get(reqUrlInfo, {
                headers: {
                'Authorization': 'Bearer ' + token
                }
            });
            if (topicInfo.data.User == null) {
                this.topicCreatorRemoved = true;
            }
            this.topic = topicInfo.data;
            this.topic.hasFollowed = Array.from(this.topic.hasFollowed).filter(char => char !== ',');
            if (this.topic.hasFollowed.includes(this.currentUser.id.toString())) {
                this.followed = true;
            }
            const config = {
                headers: {
                    'Authorization': 'Bearer ' + token
                },
                params: {
                    followers: this.topic.hasFollowed,
                }
            };
            const topicFollowers = await axios.get('/users/?limit=10', config);
            this.followers = topicFollowers.data;
            const reqUrlPosts = reqUrlInfo + '/posts/?order=recent';
            const topicPosts = await axios.get(reqUrlPosts, {
                headers: {
                'Authorization': 'Bearer ' + token
                }
            });
            if (topicPosts.data.length == 0) {
                this.noPosts = true;
            } else {
                this.posts = topicPosts.data;
            }
        }
    }
}
</script>

<style lang="scss">

.topic {
    &__body {
        &__feed {
            &__posts {
                padding: inherit;
            }
            &__publish {
                position: relative;
                &::after {
                    content: '';
                    height: 2px;
                    background-color: #D1515A;
                    position: absolute;
                    bottom: 0;
                    width: 95%;
                }
                label {
                    @media screen and (max-width: 992px) {
                        font-size: 1.5rem;
                    }
                }
                &__content {
                    min-height: 100px;
                }
                &__submit {
                    display: flex;
                    justify-content: flex-end;
                    &__btn {
                        background-color: #091F43;
                        color: #fff;
                        font-weight: bold;
                        &:hover {
                            background-color: #D1515A;
                            color: #fff;
                        }
                    }
                }
            }
        }
    }
}
</style>