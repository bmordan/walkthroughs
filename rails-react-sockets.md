# Rails with React & ActionCable (web sockets)

Remind yourself how web sockets work. Rails has wrapped this into a more intergrated service. Lets create a very simple message app.

![](https://user-images.githubusercontent.com/4499581/69637451-93200a00-1050-11ea-8596-6bd25bc84972.jpg)

Now when you send your message it is broadcast to all connected clients via Rails ActionCable.

![](https://user-images.githubusercontent.com/4499581/69637461-97e4be00-1050-11ea-8c14-a0fe50a27c6c.jpg)

## add dependencies

`rails new react-sockets --webpack=react`

`bundle add react-rails`

This adds the gem for using react in our rails app

`rails g react:install`

This is creating the assets for the build step

`rails webpacker:install:react`

Yep add the build step to webpack.

## generators

`rails g model Message body:string && rails db:migrate`

This generates the model with a single property of the body (a message string). It's followed by the command to run the migration and create the table in the database.

`rails g controller Messages`

This controller will render our react component.

`rails g react:component Chat messages:array`

OK this comes from the `react-rails` gem and enables us to create a stateful component. Notice I am passing in the props the component will receive from the controller.

`rails g channel chat`

## code samples

Create the ChatChannel class in `/app/channels/chat_channel.rb`

```ruby
class ChatChannel < ApplicationCable::Channel
  def subscribed
    stream_from "chat:chat_channel"
  end

  def onmessage(data)
    message = Message.create(body: data["message"])
    socket = { message: message.body }
    ChatChannel.broadcast_to("chat_channel", socket)
  end

  def unsubscribed
    # Any cleanup needed when channel is unsubscribed
  end
end
```

Add the controller that renders the react component in `/app/controllers/messages_controller.rb`

```ruby
class MessagesController < ApplicationController
  def index
    render component: "Chat", props: { messages: Message.all.map { |m| m.body } }
  end
end
```

Now the react components in `/app/javascript/components/Chat.js`

```js
import React, {useState} from "react"
import PropTypes from "prop-types"
import { createConsumer } from "@rails/actioncable"

const Say = ({onSay}) => {
  const [message, setMessage] = useState("")
  const onSubmit = e => {
    onSay({message})
    setMessage("")
  }
  return <form>
    <input name="msg" placeholder="add your message" onChange={e => setMessage(e.target.value)} value={message} />
    <button type="button" onClick={onSubmit}>Send</button>
  </form>
}

class Chat extends React.Component {
  state = {
    messages: this.props.messages
  }
  componentDidMount() {
    const socket = createConsumer()
    socket.subscriptions.create({
      channel: "ChatChannel"
    }, {
      connected: () => {
        console.log("Connected")
      },
      disconnected: () => {
        console.log("Disconnected")
      },
      received: data => {
        this.setState({messages: [...this.state.messages, data.message]})
      }
    })

    this.setState({
      channel: socket.subscriptions.subscriptions[0] 
    })
  }

  render () {
    return (
      <section>
        <Say onSay={message => this.state.channel.perform("onmessage", message)} />
        <hr />
        Messages: {this.state.messages.join(", ")}
      </section>
    );
  }
}

Chat.propTypes = {
  messages: PropTypes.array
}

export default Chat
```

[Back](https://github.com/bmordan/walkthroughs)
