<script setup>
import { nextTick, onBeforeMount, ref, watch } from "vue";
import { useRouter } from "vue-router";
import { getCurrentUserAuth, signOutUser } from "../services/firebase_auth.js";
import {
	messages,
	addNewMessage,
	addUserToUsersContacted,
	getUserDetailsWithEmail,
	registerMessageListener,
	incrementUnreadCounterWithContactedUser
} from "../services/firebase_firestore.js";
import MessageBubble	 from "../components/MessageBubble.vue";
import TheNavList from "../components/TheNavList.vue";
import NewChatDialog from "../components/NewChatDialog.vue";


const router = useRouter();

let currentUserAuth = null;

const sender = ref({ id: null, username: null, email: null, first_name: "", last_name: "", users_contacted: []});
const recipient = ref({ id: null, username: null, email: null, first_name: "", last_name: "", users_contacted: []});
const newMessage = ref("");
const usersContacted = ref([]);
const drawer = ref(false);


const signOutOfAccount = () => {
   console.log("HomePage.signOutOfAccount -> signing out...");

   if (signOutUser()) {
      console.log("HomePage.signOutOfAccount -> User signed out successfully.");
      router.push("/");
		location.reload();
   }
   else {
      console.error(`HomePage.signOutOfAccount -> Failed to sign out of account.`);
   }
}


const loadAllMessagesAndClearUnreadCounter = async (targetEmail = null) => {
   if (targetEmail) {
      if (targetEmail === recipient.value.email) return;

      recipient.value = await getUserDetailsWithEmail(targetEmail);
   }

	const recipientIndex = usersContacted.value.findIndex(user => user.id === recipient.value.id);
	const numOfUnreadMessages = usersContacted.value[recipientIndex].unread_counter;

   messages.value = [];
   registerMessageListener(sender.value.id, recipient.value.id);
   
   watch(messages, () => {
      nextTick(() => {
         scrollToBottomOfMessages();
      });
   }, { immediate: true, deep: true });

	await incrementUnreadCounterWithContactedUser(sender.value.id, recipient.value.id, -1 * numOfUnreadMessages);
	usersContacted.value[recipientIndex].unread_counter = 0;
}


const sendMessage = async (keyEvent = null) => {
   if (keyEvent?.shiftKey) return;

   if (!recipient.value.id) return;

   console.log(`HomePage.sendMessage() -> attempting to send a message...`);

	const currentDateAndTime = new Date().toISOString();
	const currentDateTime = new Date().toLocaleString();
	console.log(`HomePage.sendMessage() -> current date and time: ${currentDateAndTime} | ${currentDateTime}`);

   try {
      await addNewMessage(sender.value.id, recipient.value.id, newMessage.value, currentDateTime);
      newMessage.value = "";

      // if the recipient is not in the users_contacted list, add the user. This is done for both the sender and recipient user docs.
      if (!usersContacted.value.find(user => user.id === recipient.value.id)) {
         usersContacted.value.push(recipient.value);
         await addUserToUsersContacted(sender.value.id, recipient.value.id, recipient.value.username, recipient.value.email);
			await addUserToUsersContacted(recipient.value.id, sender.value.id, sender.value.username, sender.value.email);
      }
		else {
			await incrementUnreadCounterWithContactedUser(recipient.value.id, sender.value.id, 1);
		}

      console.log(`HomePage.sendMessage() -> new message sent successfully!`);
   }
   catch (error) {
      console.error(`HomePage.sendMessage() -> failed to add message doc to db. Error code: ${error.code}`);
      console.error(error);
   }

   scrollToBottomOfMessages();
}


const newChat = (newRecipientEmail) => {
	console.log(`HomePage.newChat() -> new chat with: ${newRecipientEmail}`);
	loadAllMessagesAndClearUnreadCounter(newRecipientEmail);
}


const scrollToBottomOfMessages = () => {
	window.scrollTo(0, document.body.scrollHeight);
}


onBeforeMount(async () => {
   currentUserAuth = await getCurrentUserAuth();
   sender.value = await getUserDetailsWithEmail(currentUserAuth.email);
   usersContacted.value = sender.value.users_contacted;
});
</script>


<template>
<v-app class="app">
	<v-navigation-drawer
		class="pa-2 bg-surface-container-low flex-nav-drawer"
		v-model="drawer"
		app
	>
		<template #prepend>
			<div class="pt-10 pb-5">
				<NewChatDialog
					@new-chat-with="(newRecipient) => newChat(newRecipient)"
				></NewChatDialog>
			</div>
		</template>
		<TheNavList
			:users-contacted="usersContacted"
			:recipient="recipient"
			:load-all-messages="loadAllMessagesAndClearUnreadCounter"
			:sign-out-of-account="signOutOfAccount"
		></TheNavList>
		<template v-slot:append>
			<v-btn
				class="text-none bg-error-container px-2 mb-3"
				block
				rounded="xl"
				prepend-icon="mdi-logout"
				@click="signOutOfAccount"
			>Sign out</v-btn>
		</template>
	</v-navigation-drawer>

	<v-app-bar flat>
		<v-app-bar-nav-icon
			color="smoke"
			@click="drawer = !drawer"
		></v-app-bar-nav-icon>
		<v-toolbar-title>
			{{ `${recipient.first_name} ${recipient.last_name}` }}
		</v-toolbar-title>
	</v-app-bar>

	<v-main>
		<v-container
			fluid
			fill-height
		>
			<v-row
				align="center"
				justify="center"
			>
				<v-col cols="12">
					<v-card
						v-if="!recipient.id"
						width="900px"
						color="surface"
						style="margin: auto; height: 200px"
					>
						<v-card-title>
							<h1 class="text-h2 text-center">
								Hey {{ sender.first_name }} 👋
							</h1>
						</v-card-title>
						<v-card-text style="text-align: center; min-height: 762px;">
							<NewChatDialog
								@new-chat-with="(newRecipient) => newChat(newRecipient)"
							></NewChatDialog>
						</v-card-text>
					</v-card>
					<v-card
						flat
						width="900px"
						style="margin: auto;"
					>
						<v-card-text>
							<v-list
								class="message-list"
								lines="two"
							>
								<v-list-item v-for="message in messages" :key="message.id">
									<MessageBubble
										:text-message="message.text_message"
										:user-name="message.sender_id === sender.id ? 'You' : recipient.first_name"
										:sent-on="message.sent_on"
									></MessageBubble>
								</v-list-item>
							</v-list>
						</v-card-text>

						<v-card-actions v-if="recipient.id"
							position="fixed"
							style="width: inherit"
						>
							<v-textarea
								class="new-message-textarea"
								label="New message"
								v-model="newMessage"
								variant="solo-filled"
								rows="1"
								max-rows="5"
								auto-grow
								rounded="xl"
								bg-color="surface-container-high"
								style="white-space: pre-line;"
								@keydown.enter="sendMessage($event)"
							>
								<template #append-inner>
									<v-btn
										rounded="xl"
										color="white"
										:disabled="newMessage.trim() === ''"
										append-icon="mdi-send-outline"
										@click="sendMessage()"
									></v-btn>
								</template>
							</v-textarea>
						</v-card-actions>
					</v-card>
				</v-col>
			</v-row>
		</v-container>
	</v-main>
</v-app>
</template>



<style scoped>
.app {
	max-width: 100vw;
	max-height: 100%;
}

.new-message-textarea {
	position: fixed;
	bottom: 0;
	width: inherit;
	z-index: 1000;
}
</style>