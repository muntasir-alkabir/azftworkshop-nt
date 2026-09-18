# Exercise 2: 🎛️ Deploy a model

**⏱️ ~8 minutes**

You need two models: one that **reasons and writes** (chat), and one that **turns text into vectors** (embedding) so your catalogue can be searched by meaning.

This is **model strategy** in practice — including the deployment-name indirection that makes swapping models a config change rather than a project.

## ✅ Outcome

- A chat deployment named `gpt-chat`
- An embedding deployment named `text-embed`
- A successful test call in the playground

---

### Task 2.1: Deploy the chat model

1. In your project, select **Build** in the top navigation, then **Deployments** in the left pane. Click **Deploy a base model**.

2. The model catalogue opens. Search for a current **GPT-class chat model** — a `mini`-class one is the right default — and select it. Check the **Type** reads *Chat completion* before you continue.

    ![Step 2](../media/s1-lab1-ex2-02.png)

    > 💡 Any current GPT-class chat model works for this lab. Pick on *sufficiency*, not leaderboard rank. A `mini`-class model is the right default here: cheaper, faster, and easily good enough for a catalogue Q&A agent.

3. Click **Deploy** and choose **Custom settings** (not *Default settings* — you need to control the name and the rate limit). Then set:

    | Field | Value |
    |---|---|
    | Deployment name | `gpt-chat` |
    | Deployment type | **Global Standard** (pay-as-you-go) |
    | Tokens per minute rate limit | `30K` |

    ![Step 3](../media/s1-lab1-ex2-03.png)

    > ⚠️ Use the **deployment name `gpt-chat`**, not the model's own name. Everything downstream — agent, code, guardrails — binds to this name, so you can swap the underlying model later without touching any of them.

    > 🛡️ Note the **Guardrails** field, defaulted to `DefaultV2`. Content safety is attached at *deployment* time, not bolted on later. You will inspect and extend this policy in [Lab 2, Exercise 2](../Lab%202%20-%20Secure,%20evaluate%20&%20monitor%20your%20first%20agent/Exercise%202%20-%20Guardrails%20and%20content%20safety.md).

4. Click **Deploy** and wait for the **Deployment status** to reach **Succeeded**.

    ![Step 4](../media/s1-lab1-ex2-04.png)

    > 🔍 Look at the **Call this model** sample code in the details pane. It reads `deployment_name = "gpt-chat"` — the model name appears nowhere. That is the indirection from step 3, doing its job.

---

### Task 2.2: Deploy the embedding model

1. Return to **Build → Model → Deployments** and click **Deploy a base model** again.

2. Search the catalogue for `text-embedding` and select a current embedding model. Confirm the type reads **Embeddings**.

    > 💡 A `small` embedding model is the right default for a product catalogue — the documents are short and the vocabulary is narrow. Reach for a `large` model only when retrieval quality measurably demands it.

3. Choose **Deploy → Custom settings**. Set the **Deployment name** to `text-embed`, leave the type as **Global Standard**, and set the rate limit to `30K` TPM. Click **Deploy**.

    > 🔑 The embedding model used to **build** the index must be the same one used to **query** it. Changing it later means a full re-index — decide once, deliberately.

4. Confirm both deployments now show **Succeeded** on the **Deployments** list.

    ![Step 4](../media/s1-lab1-ex2-08.png)

---

### Task 2.3: Smoke-test the chat deployment

1. From the deployment details pane, click **Open in playground**.

2. Check the **Model** selector at the top reads **`gpt-chat`** — your deployment name, not the model name. open the playground or double click

3. Send this prompt:

    ```text
    In two sentences, explain the difference between grounding and fine-tuning.
    ```

    > 💡 Look under the response: the playground reports the **model, latency and token count** for every turn. Note the token number — you will use exactly this signal for cost work in [Lab 2, Exercise 5](../Lab%202%20-%20Secure,%20evaluate%20&%20monitor%20your%20first%20agent/Exercise%205%20-%20Cost%20monitoring%20and%20FinOps.md).

4. Now ask a question the model cannot possibly know:

    ```text
    What is Contoso Outdoors' return window for the Summit 2P tent?
    ```

    **First, check the Tools panel.** New playgrounds often have **Web search** attached by default. Run the question **with it on**, then **remove the Web search tool**, start a new chat, and run the same question again.

    > 🔧 If your playground has **no** Web search tool, add it from the Tools panel for the first run, then remove it for the second. The comparison is the point — you need both answers.

    ![Step 4](../media/s1-lab1-ex2-12.png)

    Record both answers:

    | Tools | Answer | Tokens | Latency |
    |---|---|---|---|
    | Web search **on** | | | |
    | No tools | | | |

    > 👀 **This is the most important observation in Lab 1.** With web search on, the model returns a confident, specific return window — grounded on whatever the search happened to surface, which has nothing to do with Contoso. With no tools, a current model will usually **admit it doesn't know**. So the failure mode is not simply "models make things up" — it is that **a model grounded on the wrong source is more dangerous than one with no source at all**, because it sounds sourced.

    > 💰 Compare the token counts. In our run the web-search answer cost **~62× the tokens and 7× the latency** of the honest "I don't know" — and it was the wrong answer. Retrieval is never free, and unfocused retrieval is expensive *and* harmful.

    > 🎯 Exercise 3 fixes this properly: not by removing retrieval, but by pointing it at an **approved, scoped** corpus.

---

## 🧾 Checkpoint

- [ ] `gpt-chat` deployment — status **Succeeded**
- [ ] `text-embed` deployment — status **Succeeded**
- [ ] Playground returns a response from `gpt-chat`
- [ ] You have recorded the Summit 2P answer **both with and without** the Web search tool, including token counts

---

## 🧠 What we learned

- **Deployment names are an abstraction layer** — bind everything to the name, keep the model swappable. The generated sample code proves it: it never mentions the model.
- **Global Standard vs provisioned** is a cost/predictability trade-off; standard is right for dev.
- **Guardrails attach at deployment time**, not as an afterthought.
- The **embedding model is a long-lived commitment** — changing it forces a re-index.
- **Check which tools are attached by default.** A model grounded on an irrelevant source gives a confident wrong answer, at many times the cost of admitting it doesn't know. That is the problem the next exercise solves — with *scoped* grounding.

---

**Previous:** [◀️ Exercise 1](./Exercise%201%20-%20Provision%20your%20Foundry%20project.md) · **Next:** [Exercise 3 — Ground your agent with knowledge ▶️](./Exercise%203%20-%20Ground%20your%20agent%20with%20knowledge.md)
