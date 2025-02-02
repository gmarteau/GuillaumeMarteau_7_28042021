<template>
    <div class="postDetails">
        <section class="postDetails__post row">
            <Post 
                :id="post.id"
                :authorId="post.UserId"
                :imageUrl="post.User.profilePicture"
                :firstName="post.User.firstName"
                :lastName="post.User.lastName"
                :username="post.User.username"
                :topicId="post.TopicId"
                :topic="post.Topic.name"
                :content="post.content"
                :numberOfLikes="post.likes"
                :numberOfDislikes="post.dislikes"
                :hasLiked="post.hasLiked"
                :hasDisliked="post.hasDisliked"
                :numberOfComments="post.numberOfComments"
                @post-deleted="redirectToTopicPage"
                @post-updated="refreshPost"
                @post-liked="refreshPost"
                @post-disliked="refreshPost"
            />
        </section>

        <section class="postDetails__comments">
            <h1 class="postDetails__comments__title h4">Commentaires</h1>

            <div class="postDetails__comments__body row">
                <div class="postDetails__comments__publish container py-3">
                    <b-form id="publishNewCommentForm" class="row" @submit.stop.prevent="publishNewComment" novalidate>
                        <b-form-group id="newCommentGroup" class="col-10 col-md-11 col-lg-10 mb-0">
                            <b-form-textarea id="newCommentInput" name="newCommentInput" v-model="$v.form.newComment.$model" :state="validateState('newComment')" aria-describedby="newCommentInputFeedback" aria-label="Ecrire un commentaire" type="text" rows="2" placeholder="Ecrivez un commentaire..." required></b-form-textarea>
                            <b-form-invalid-feedback id="newCommentInputFeedback" :state="validateState('newComment')">Votre commentaire ne peut pas être vide</b-form-invalid-feedback>
                        </b-form-group>
                        <div class="postDetails__comments__publish__submit col-2 d-none d-lg-block">
                            <button type="submit" class="postDetails__comments__publish__submit__btn btn px-3 py-1">
                                Commenter
                            </button>
                        </div>
                        <div class="postDetails__comments__publish__submit col-2 col-md-1 d-lg-none pl-0">
                            <button type="submit" class="postDetails__comments__publish__submit__btn btn px-2 py-1" aria-label="Commenter">
                                <i class="fas fa-paper-plane"></i>
                            </button>
                        </div>
                    </b-form>
                </div>

                <Comment
                    v-for="comment in comments"
                    :key="comment.id"
                    :id="comment.id"
                    :authorId="comment.UserId"
                    :imageUrl="comment.User.profilePicture"
                    :firstName="comment.User.firstName"
                    :lastName="comment.User.lastName"
                    :username="comment.User.username"
                    :content="comment.content"
                    :numberOfLikes="comment.likes"
                    :numberOfDislikes="comment.dislikes"
                    :hasLiked="comment.hasLiked"
                    :hasDisliked="comment.hasDisliked"
                    @comment-deleted="refreshComments"
                    @comment-updated="refreshComments"
                    @comment-liked="refreshComments"
                    @comment-disliked="refreshComments"
                />
            </div>
        </section>
    </div>
</template>

<script>
import axios from 'axios'
import Post from '../components/Post'
import Comment from '../components/Comment'
import { mapGetters } from 'vuex'
import { validationMixin } from 'vuelidate'
import { required } from 'vuelidate/lib/validators'

export default {
    name: 'PostDetails',
    mixins: [validationMixin],
    components: {
        Post,
        Comment
    },
    data() {
        return {
            post: '',
            comments: [],
            form: {
                newComment: ''
            }
        }
    },
    validations: {
        form: {
            newComment: {
                required
            }
        }
    },
    computed: {
        ...mapGetters(['currentUser', 'loggedIn'])
    },
    methods: {
        validateState(field) {
            const { $dirty, $error } = this.$v.form[field];
            return $dirty ? !$error : null;
        },
        async publishNewComment() {
            const token = localStorage.getItem('token');
            this.$v.form.$touch();
            if (this.$v.form.$anyError) {
                return;
            }
            const url = window.location.search;
            const searchUrl = new URLSearchParams(url);
            const topicId = searchUrl.get('topic');
            const postId = searchUrl.get('id');
            const reqUrlPost = '/topics/' + topicId + '/posts/' + postId + '/comments';
            const createComment = await axios.post(reqUrlPost, {
                userId: this.currentUser.id,
                content: this.form.newComment
            }, {
                headers: {
                'Authorization': 'Bearer ' + token
                }
            });
            console.log(createComment.data);
            const reqUrlGet = reqUrlPost + '/?order=recent';
            const commentsRefreshed = await axios.get(reqUrlGet, {
                headers: {
                'Authorization': 'Bearer ' + token
                }
            });
            this.comments = commentsRefreshed.data;
            this.form.newComment = '';
        },
        redirectToTopicPage() {
            this.$router.push({ path: '/topic', query: { id: this.post.TopicId } });
        },
        async refreshComments() {
            const token = localStorage.getItem('token');
            const url = window.location.search;
            const searchUrl = new URLSearchParams(url);
            const topicId = searchUrl.get('topic');
            const postId = searchUrl.get('id');
            const reqUrl = '/topics/' + topicId + '/posts/' + postId + '/comments/?order=recent';
            const commentsRefreshed = await axios.get(reqUrl, {
                headers: {
                'Authorization': 'Bearer ' + token
                }
            });
            this.comments = commentsRefreshed.data;
        },
        async refreshPost() {
            this.post = '';
            const token = localStorage.getItem('token');
            const url = window.location.search;
            const searchUrl = new URLSearchParams(url);
            const topicId = searchUrl.get('topic');
            const postId = searchUrl.get('id');
            const reqUrl = '/topics/' + topicId + '/posts/' + postId;
            const postRefreshed = await axios.get(reqUrl, {
                headers: {
                'Authorization': 'Bearer ' + token
                }
            });
            this.post = postRefreshed.data;
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
            const topicId = searchUrl.get('topic');
            const postId = searchUrl.get('id');
            const reqUrlPost = '/topics/' + topicId + '/posts/' + postId;
            const post = await axios.get(reqUrlPost, {
                headers: {
                'Authorization': 'Bearer ' + token
                }
            });
            this.post = post.data;
            const reqUrlComments = reqUrlPost + '/comments/?order=recent';
            const comments = await axios.get(reqUrlComments, {
                headers: {
                'Authorization': 'Bearer ' + token
                }
            });
            this.comments = comments.data;
        }
    }
}
</script>

<style lang="scss">

.postDetails {
    width: 75%;
    margin: auto;
    @media screen and (max-width: 992px) {
        width: 100%;
    }
    &__comments {
        &__body {
            border: solid 1px #d3d3d3;
            border-radius: .3rem;
        }
        &__publish {
            min-width: 100%;
            background-color: #D1515A;
            .invalid-feedback {
                color: #fff;
            }
            &__submit {
                display: flex;
                justify-content: center;
                align-items: center;
                &__btn {
                    background-color: #091F43;
                    color: #fff;
                    font-weight: bold;
                    &:hover {
                        background-color: #fff;
                        color: #091F43;
                        border: solid 2px #091F43;
                    }
                }
            }
        }
    }
}
</style>