# Exercise 3: 📚 Ground your agent with knowledge

**⏱️ ~15 minutes**

An ungrounded model invents. A model grounded on the *wrong* source invents with a citation, which is worse — you saw both in Exercise 2.

In this exercise you give the agent a **universe of truth**: the approved Contoso Outdoors catalogue, stored where an enterprise would actually store it, and indexed for retrieval.

This is **enterprise data grounding and governance** in practice.

## ✅ Outcome

- The catalogue in Blob Storage, in your own resource group
- An Azure AI Search resource connected to the project
- A knowledge base named `contoso-catalogue`, indexed with `text-embed` and reporting **Active**

---

### Task 3.1: Put the catalogue somewhere real

Foundry does not host your documents. Knowledge sources point at systems that already hold the data — Blob Storage, SharePoint, Azure SQL, Fabric, or an existing search index. For this lab you will use Blob Storage.

1. In the Azure portal, create a **Storage account** in `rg-genai-workshop-<initials>`:

    | Field | Value |
    |---|---|
    | Name | `stgenaiworkshop<initials>` (lowercase, no hyphens) |
    | Region | *your lab region* |
    | Primary service | **Azure Blob Storage or Azure Data Lake Storage** |
    | Performance | Standard |
    | Redundancy | **Locally-redundant storage (LRS)** |

    ![Storage account settings](../media/s1-lab1-ex3-01.png)

    > 💰 The default redundancy is **GRS**, which replicates to a second region and costs roughly twice LRS. For disposable lab data that is waste. Redundancy is a per-workload decision — match it to how bad it would be to lose the data, not to the default.

2. Open the account, click storage browser from left, go to **Containers**, and add a container named `contoso-catalogue`. Leave the access level **Private**.

3. **Grant yourself blob data access before you try to upload.** On the storage account, open **Access control (IAM) → + Add → Add role assignment**, choose **Storage Blob Data Contributor**,Select members and assign it to your own user.

    > ⚠️ **Owner is not enough, and this catches almost everyone.** Owner and Contributor are **control-plane** roles: they let you create and configure the storage account but grant **no access to the data inside it**. Uploading a blob is a *data-plane* operation and needs a `Storage Blob Data *` role.
    >
    > You may not notice, because the portal will quietly fall back to the account access key if your tenant allows it. Many enterprise tenants **disable shared-key auth by policy** — and increasingly do so automatically minutes after an account is created. When that happens the upload fails with *"You do not have the required permissions"* even though you own the subscription. Grant the role now and the question never comes up.
    >
    > This is the same lesson as Task 3.2 below, one layer earlier: **in a keyless architecture, every hop needs its own explicit grant.**

4. Open the container, click **Upload**, and upload the **20 markdown files** from the [`contoso-outdoors-catalogue`](../lab-data/contoso-outdoors-catalogue/README.md) package — everything **except** `README.md`. Confirm the container shows **20 items**.

    ![Blobs uploaded](../media/s1-lab1-ex3-02.png)

    > ⚠️ **Leave `README.md` out, deliberately.** That file is the package's own documentation, and it contains a table of the correct answers — the Summit 2P price, its waterproof rating, and a note that the 2023 price list is superseded. Index it and your agent can pass every test in Exercise 4 by quoting the answer key, while citing documents it never actually read.
    >
    > This is not housekeeping. It is the first instance of a rule this workshop returns to in [Lab 2 Exercise 2](../Lab%202%20-%20Secure,%20evaluate%20&%20monitor%20your%20first%20agent/Exercise%202%20-%20Guardrails%20and%20content%20safety.md#task-24-write-down-what-is-not-covered): **for a RAG agent, your trust boundary is every document you index.** A file that is harmless on disk becomes authoritative the moment it is in the corpus. Curating what goes in *is* a security control.

    > 📦 The package deliberately includes **one superseded 2023 price list** that disagrees with the current one on every product. Leave it in. Real corpora contain stale documents, and you will use this one in Lab 2.

---

### Task 3.2: Connect a search resource

1. Back in Foundry, open **Build → Knowledge**. If no search resource is connected, click **Create new resource**, place it in your resource group, and click **Create**.

    > ⚠️ **Set Auth Type to `Project Managed Identity`.** The connection dialog defaults to **API Key**. If you leave it, the three role assignments below do nothing — the project authenticates with a shared key and the whole keyless story in this lab quietly stops being true. Pick managed identity, which is what the rest of this exercise assumes.

    > ⚠️ **Azure AI Search is frequently at capacity.** If you see *"This region is at capacity"*, pick another region — the search resource does **not** have to match your project's region. Note which region you ended up in; cross-region retrieval adds latency you would measure before shipping.

    > 💰 Unlike the models, which bill per call, a search resource bills **per hour whether or not you query it**. It is the one resource in this lab that costs money while you sleep. Delete it in the [clean-up](../README.md#-clean-up).

    > 🧰 **Reusing a search service you created earlier?** Two extra steps, both invisible until they fail:
    >
    > - A search service created outside Foundry defaults to **API-key-only** data-plane auth, so *Project Managed Identity* fails with the unhelpful *"Failed to fetch knowledge bases for connection…"*. Fix with:
    >   ```bash
    >   az search service update --name <search> --resource-group <rg> \
    >     --auth-options aadOrApiKey --aad-auth-failure-mode http403
    >   ```
    > - Its identity also needs **Storage Blob Data Reader** on your storage account, in addition to the three grants below. Without it the indexer refuses to be created at all: *"Unable to retrieve blob container … granted permission (e.g., Storage Blob Data Reader)"*.
    >
    > A search resource that **Foundry creates for you** needs neither — it brokers blob access itself. This is the difference between the happy path and a bring-your-own-resource path, and it is worth knowing before you meet it in production.

2. Once connected, click **Create a knowledge base** and set:

    | Field | Value |
    |---|---|
    | Name | `contoso-catalogue` |
    | Chat completions model | `gpt-chat` |
    | Retrieval reasoning effort | Minimal |
    | Output mode | Extractive data |

3. **Fix the permissions now, before you index.** This is the single most common reason this lab fails, and the errors it produces are misleading. Grounding needs **three** things that Foundry does not configure for you:

    | # | Where | What | Why |
    |---|---|---|---|
    | 1 | Search service → **Identity** | Turn **System assigned** status to **On** | Foundry creates the search service **without** an identity, yet its own warning depends on one |
    | 2 | Foundry resource → **Access control (IAM)** | Grant **Cognitive Services User** to the *search service's* identity | Lets search call `text-embed` to vectorise queries |
    | 3 | Search service → **Access control (IAM)** | Grant **Search Index Data Reader** to the *Foundry resource and project* identities | Lets the agent read the knowledge base |

    > 🔑 Note the direction: **2 and 3 point opposite ways.** Search must call your models, *and* your agent must call search. A keyless architecture means every hop needs its own grant — that is the real cost of "no secrets in code", and it is worth understanding before you meet it in production.

    > ⚠️ **Be clear about what you just built.** Grant 3 gives the *agent's* identity blanket read access to the whole index. Every user gets identical results, because the search is performed as the service, not as the person asking. That is fine here — the Contoso catalogue is public product content with nothing to trim. It is **not** fine for SharePoint, HR data, or anything with per-user permissions, where it is a well-known permission-bypass anti-pattern. Production needs **query-time trimming against the calling user's identity** — you carry `acl_ids` on each chunk and filter on them, or use a connector that propagates the user's token. You will record this as a gap in [Lab 2 Exercise 1](../Lab%202%20-%20Secure,%20evaluate%20&%20monitor%20your%20first%20agent/Exercise%201%20-%20Identity,%20networking%20and%20landing%20zone%20hardening.md).

    > ⏳ Role assignments take **2–5 minutes** to propagate. If you enable the identity in step 1, wait a moment before step 2 — the identity will not appear in the picker until Entra has registered it.

    > 🩺 **Symptom decoder**, because neither error names the missing role:
    >
    > | Error | Missing |
    > |---|---|
    > | `403 Forbidden` enumerating tools on `…/knowledgebases/…/mcp` | #3 |
    > | `502 Bad Gateway` — *"failed to authenticate to the vectorization endpoint"* | #1 or #2 |

---

### Task 3.3: Add the catalogue as a knowledge source

1. Under **Knowledge sources**, click **Add sources → Azure Blob Storage**, and configure:

    | Field | Value |
    |---|---|
    | Name | `contoso-catalogue-blob` |
    | Storage account | `stgenaiworkshop<initials>` |
    | Container name | `contoso-catalogue` |
    | **Authentication type** | **`System assigned identity`** |
    | Content extraction mode | Minimal |
    | **Embedding model** | **`text-embed`** |

    ![Blob knowledge source configuration](../media/s1-lab1-ex3-03.png)

    > ⚠️ **Authentication type defaults to `API Key` here too.** Same trap as the connection dialog, one layer down: leave it and the indexer reads your blobs with a shared key rather than the identity you spent Task 3.2 configuring. Set it to **System assigned identity**.

    > 🔑 **This is the binding that matters.** The embedding model here builds the vectors, and the same deployment must serve every future query. Change it later and you are not tuning retrieval — you are invalidating the whole index. This is why Exercise 2 insisted on the stable name `text-embed` rather than the model's own name.

    > 💡 **Content extraction mode** is where chunking lives now. *Minimal* is right for short, well-structured markdown like this catalogue. Richer modes cost more per document and pay off on long PDFs with tables and images. Tune it against an evaluation set, never by intuition.

2. Click **Create**. The source appears with status **Creating** while it indexes.

    ![Source attached and indexing](../media/s1-lab1-ex3-04.png)

3. Click **Save knowledge base**.

4. Wait for the source status to become **Active**, then **verify the index actually has content**. In the Azure portal, open your search service → **Search management → Indexes** and check the **Document count**.

    ![Index document count](../media/s1-lab1-ex3-06.png)

    > 💡 **Expect at least 20 — and never 0.** The index counts *chunks*, not files, so the number can exceed your file count when a document is long enough to split. Anything at or slightly above your file count is normal and healthy. A count of **0** is not.
    >
    > *(The screenshot shows 22 from an earlier run that also indexed the package README and a longer document that split. Your exact number will differ — the shape is what matters, not the digit.)*

    > ⚠️ **"Active" does not mean "populated".** The source status describes the *connection*, not the data. If permissions were wrong when the indexer first ran, you get an Active source over an **empty index** — and the agent will politely tell every user it doesn't know anything. A document count of **0** is the tell.

5. If the count is 0, the first indexer run failed. Fix the roles in Task 3.2, then in the portal open your search service → **Search management → Indexers**:

    1. Select the indexer and read **Docs succeeded** — `0/20` confirms it.
    2. Click **Reset**, then **Run**.

    > 🔑 **Reset is not optional.** Indexers track which blobs they have already seen, and a plain **Run** skips unchanged files — so it would do nothing. **Reset** clears that state and forces a full re-index. Foundry's own UI has no re-sync button, so this is the recovery path.

---

## 🧾 Checkpoint

- [ ] Storage account created, container `contoso-catalogue` holds **20 blobs** — `README.md` deliberately excluded
- [ ] **Storage Blob Data Contributor** granted to your own user on the storage account
- [ ] Search resource connected to the project, **Auth Type = Project Managed Identity** — note its region
- [ ] Search service **system-assigned identity is On**
- [ ] Search identity has **Cognitive Services User** on the Foundry resource
- [ ] Foundry identities have **Search Index Data Reader** on the search service
- [ ] Knowledge base `contoso-catalogue` saved, source reports **Active**
- [ ] **Index document count is at least 20, and not 0** — Active alone is not proof
- [ ] The embedding model on the source reads `text-embed`

---

## 🧠 What we learned

- Foundry **does not store your documents**. Grounding points at systems of record, which is why data governance stays where the data already lives.
- Grounding quality is decided at **ingest and extraction time**, long before the model is called.
- The **embedding model binds the index** — index and query must match, and changing it means a rebuild.
- Retrieval infrastructure has a **standing hourly cost**, unlike per-call model billing. It is the line item people forget.
- **Keyless is not free.** Removing shared secrets replaces one credential with a set of directional role assignments, and the failure messages point at symptoms rather than the missing grant. Budget time for identity, in the lab and in production.
- **Green status is not evidence.** An Active source over an empty index looks healthy and answers "I don't know" to everything. Verify the document count — the habit that saves you here is the same one Lab 2 formalises as evaluation.

---

**Previous:** [◀️ Exercise 2](./Exercise%202%20-%20Deploy%20a%20model.md) · **Next:** [Exercise 4 — Build and test your agent ▶️](./Exercise%204%20-%20Build%20and%20test%20your%20agent.md)
