```rs

create_server_function_macro!(my_server_function, [SessionMiddleware]);

pub trait Syncable {
    async fn fetch(id: Uuid) -> Self;
}

#[derive(Clone)]
pub struct Synced<T: Syncable+Clone> {
    data: Arc<RwLock<T>>,
    loading: Arc<RwLock<bool>>
}

impl<T: Syncable> Synced<T> {
    pub fn modify(&mut self, f: Fn(T) -> impl Future<Output = T>) {
        let data = self.data.clone();
        let loading = self.loading.clone();
        tokio::spawn(move || async {
            *loading.write() = true;
            let data_modified = f(data.read().clone()).await;
            *data.write() = data_modified;
            *loading.write() = false;
        });
    }

    pub fn modify_sync(&mut self, f: Fn(T) -> T) {
        let new_data = f(self.data.read().clone());
        *self.data.write() = new_data;
    }

    pub fn read(&self) -> T {
        self.data.read().clone()
    }

    pub fn loading(&self) -> bool {
        *self.loading.read()
    }
}


#[derive(Clone)];
pub struct ChatMessage {
    owner: Uuid,
    content: String
}

#[derive(Clone)]
struct Chat {
    messages: Vec<ChatMessage>
}

impl Chat {
    #[my_server_function]
    pub async fn send_message(&mut self, new_message: String, session: Session) {
        self.messages.push(ChatMessage { owner: session.owner, content: new_message });
    }
}


#[derive(Clone)];
struct ChatViewProps {
    id: Uuid,
}

#[derive(Clone)]
struct ChatView {
    chat_sync: Synced<Chat>,
    new_message: String
}

impl ChatView {
    pub fn send_message(&mut self) {
        let new_message = self.new_message.replace(String::new());
        self.chat_sync.modify(move |chat: &mut Chat| chat.send_message(new_message));
    }
}

impl Component for ChatView {
    async fn startup(props: ChatViewProps) -> ChatView {
        Self {
            Chat::new_synced(props.id).await
        }
    }
    fn render(self) {
        rsx! {
            Div {
                class: "flex gap-3"<
                for message in &chat.messages {
                    Div {
                        {message.content}
                    }
                }
            }
            Form {
                on_submit: |&mut self, e| {
                    self.send_message();
                }
                Input {
                    placeholder: "Hello how are you?",
                    on_change: |&mut self, e| {
                        self.new_message = e.value();
                    }
                }
                Button {
                    disabled: chat.loading()
                    r#type: "submit",
                    "Send"
                }
            }
        }
    }
}

```
