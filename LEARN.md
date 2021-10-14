# Building a platform to help people launch their NGOs on Near protocol
Hi there! In the last quest we wrote a simple voting smart contract on top of Near protocol, interacted with it and deployed it on a test network. Now let’s try building something more complex and useful.

Let’s talk about NGOs. There are approximately 10 million NGOs that are operating worldwide. They receive huge amounts of money as donations through various channels who wish to aid them for the cause they support. The problem with middlemen gobbling up the money, cases of licit NGOs and internal authorities involved in the misappropriation and mis accounting of funds has caused NGOs to lose their reputation and respect, with more and more people becoming increasingly hesitant to donate money even for the sake of humanity.

This sector is deficient of a system that traces the source of the donation, manages funds appropriately, tracks whether the donation reaches the intended recipient or not. Basically an open system where the information within can be accessed publicly. NGOs require transparent and trusted systems like Blockchain to bring back openness and reliability in the process.

Let’s build a platform where people can come and register their NGOs, in those NGOs anyone can create projects and add cause with fund requirement details. Then donors can come to check out a list of projects from a specific NGO and choose to fund any project they wish to.

We are going to use the same project structure that we defined on the previous quest so if you haven’t completed Quest 1 on NEAR (https://questbook.app), please go ahead and complete that first.
## Data model of NGO
Let’s get our data model ready for registration of NGO. First of all we would need to maintain a record of all the NGOs. Now each NGO will have some attributes of its own, like - a unique identifier (id), name and accountId (address). To model this we will be using the below class definition.

Fill in below code in your `assembly/index.ts` file.

```

import { context, PersistentUnorderedMap } from "near-sdk-as";

@nearBindgen

export class NGO {

  id: u32;

  name: string;

  address: string;

  constructor(_id: u32, _ngoName: string) {

    this.id = _id;

    this.name = _ngoName;

    this.address = context.sender;

  }

}

const ngoList = new PersistentUnorderedMap("n");

```

We are using `u32` i.e, unsigned integers for storing our ids because we know that they are never going to be negative.

`@nearBindgen` in above code is a decorator, a decorator is a function which returns another function after adding some more logic to it. In this case it converts class objects to bytes (serializes the data structure) making it possible to pass data from AssemblyScript to blockchain storage. We can’t just store AssemblyScript data models directly on blockchain because it will cause problems with cross language compatibility. One can write contracts in various languages like AssemplyScript and Rust - we need to make sure the data models are consistent across each language, requiring us to serialize. 

Also notice that we are using `PersistentUnorderedMap` here because `PersistentUnorderedMap` gives us ability to fetch all the keys in collection with `PersistentUnorderedMap.keys()` which will come in handy while fetching list of all NGO ids and also for checking out number of NGOs registered using `PersistentUnorderedMap.keys().length`.

With above class, we can create an NGO object and then append the object to `ngoList` where we are maintaining the list of all the NGOs that have been created thus far:

```

const ngo = new NGO(0, ”Child Ngo”);

ngoList.set(0, ngo);

```

On creating a new instance of NGO, i.e, by executing `new NGO(0, “Child Ngo”)` the constructor gets invoked setting up the attributes to the NGO object

Now let’s create our function skeleton for registering and fetching a list of NGOs. For registering an NGO we just need to pass on the NGO name, and the function should return newly registered NGO’s id. Whereas fetching a list of NGOs would give us a list of ids for all NGOs. Fill in below code in your `assembly/index.ts` file.

```

export function registerNGO(ngoName: string):u32 {

  return 0

}

export function getNGOs():Array {

  return \[\]

}

```

What we have done now is created a structure of how our functions will look like. We will fill these up with logic in a while.

If you recollect you can quickly test these functions out by 

```

$ yarn asb

$ near dev-deploy --wasmFile build/release/\*.wasm

$ near call  function_name ‘{“param1”: ”value1”}’ --accountId 

```
## Implementing NGO registration functionality
Finally let’s write the logic for `registerNGO` and `getNGOs`.

For registering an NGO we need to create an instance of the NGO class, and push the object along with the id to `ngoList`. Let’s write a function which will just do that and then return id for the newly created NGO.

```

export function registerNGO(ngoName: string):u32 {

  const ngo = new NGO(ngoList.keys().length, ngoName);

  ngoList.set(ngo.id, ngo);

  return ngo.id;

}

```

`getNGO` will straightforwardly return the keys of our list `ngoList`, as we just need to return the id of all NGOs.

```

export function getNGOs():Array {

  return ngoList.keys();

}

```

Now we will follow the same two step process (data modelling and contract implementation) for adding projects to a NGO.
## Data model for allocating projects
One NGO can have multiple projects running under them. Each project can raise funds under this NGO’s brand name.

Similar to NGO objects, we will also need a Project class object to define all the attributes related to a project and then we will have a list of projects as well. We also need to store mapping of NGO with a list of projects allocated to it, so that we can keep a track of which project was allocated to which NGO and also fetch a list of projects under a specific NGO (we store this information using `PersistentMap` where key would be NGO id and value would be Array of project ids under that NGO. If you are confused about when to use PersistentMap vs PersistentUnorderedMap, check out their implementations [here](https://github.com/near/near-sdk-as/blob/f3707a1672d6da6f6d6a75cd645f8cbdacdaf495/sdk-core/assembly/collections/persistentMap.ts) and [here](https://github.com/near/near-sdk-as/blob/f3707a1672d6da6f6d6a75cd645f8cbdacdaf495/sdk-core/assembly/collections/persistentUnorderedMap.ts), PersistentUnorderedMap gives some extra functionalities and the way of storing is a bit different. Ex: You won’t get functionality to fetch `keys()` on PersistentMap but you will get that in PersistentUnorderedMap. It is advised to use PersistentMap wherever applicable because it takes less memory in comparison to PersistentUnorderedMap.

Update `assembly/index.ts` with below code:

```

import { context, PersistentUnorderedMap, u128, PersistentMap } from "near-sdk-as";

@nearBindgen

class Project {

    id: u32;

    address: string;

    name: string;

    funds: u128;

    constructor(_id: u32, _address: string, _funds: u128, _name: string) {

        this.id = _id;

        this.address = _address;

        this.name = _name;

        this.funds = _funds;

    }

}

const projects = new PersistentUnorderedMap("p");

const ngoProjectMap = new PersistentMap>("np");

```

It’s very similar to how we defined class for NGO, it’s just the attributes that are a bit different. Here we are storing a mapping of NGO id with a list of projects in the `ngoProjectMap` variable.

Now let’s create our function skeleton for allocating the project and fetching a list of projects under a NGO. For adding a project we need to pass details of the project: ngo id, fund allocation address, name, funding required and the function should return newly created project id. Whereas fetching a list of Projects we need to pass NGO id and the function would give us a list of ids for all the projects under that NGOs. Fill in below code in your `assembly/index.ts` file.

```

export function addProject(ngoId: u32, address: string, name: string, funds: string):u32 {

  return 0

}

export function getProjects(ngoId: u32):Array {

  return \[\]

}

```

Notice that we are taking funds as string in addProject however storing it as u128 in Project object, this is because AssemblyScript does not support 128 bytes uint by default, hence we pass string as input and then convert it to u128 that we pull in from near-sdk.
## Implementing Project allocation
For adding a project to an NGO, we need to create an instance of Project and push it to the `projects` variable and also assign the new project id in `ngoProjectMap`. That way, we’ll be able to allow people to start sending in money for each of the projects listed under an NGO.

There are a couple of other things as well we would want to do in our addProject function, we need to check if the specified NGO exists, we can perform that check using the `assert` function. Then consider the case where the very first project is getting allotted to an NGO, in that case we want to create a new Array with this new project id and assign it to the NGO.

I think you have got enough experience now to be able to write this function on your own, try it out for a few minutes before checking out our implementation.

Even if you were not able to come up with it, no problem check out the code below and put it into our `assembly/index.ts` file

```

export function addProject(ngoId: u32, address: string, name: string, funds: string):u32 {

  assert(ngoList.contains(ngoId), "NGO not found");

  const funds_u128 = u128.from(funds);

  const newProject = new Project(projects.keys().length, address, funds_u128, name);

  projects.set(newProject.id, newProject);

  let projectIds: Array;

  if(ngoProjectMap.contains(ngoId)) {

    projectIds = ngoProjectMap.getSome(ngoId);

  } else {

    projectIds = \[\];

  }

  projectIds.push(newProject.id);

  ngoProjectMap.set(ngoId, projectIds);

  return newProject.id;

}

```

Now for fetching list of projects for a specific NGO, we can simply use `ngoProjectMap.getSome()` and return the result

```

export function getProjects(ngoId: u32):Array {

  const projectList = new Array();

  return ngoProjectMap.getSome(ngoId);

}

```
## Accepting donations
Now that our NGO registration and project allocations are up and running, it's time to accept donations. This is the most interesting part to implement.

Let’s write our function definition and unit test before the actual implementation.

The function would receive NGO id and project id as parameters and return a string saying “done”.

One point to notice here:

NEAR token is the native currency used in Near blockchain i.e,  for each and every transaction on the blockchain you have to spend NEAR tokens if you are changing or adding to the storage (this is called gas fee). You can also send deposits to any smart contract functions, we will be doing the same for making donations. 

Smallest unit of NEAR token is called yoctoNEAR, where 1 NEAR = 10^24 yoctoNEAR.

Finally let’s implement our function `donate()`, first of all we would want to put up a few checks like the NGO must be valid, the project must exist in that NGO, the deposit amount should be less than the required fund by project, and then we would pull out the address from the Project object and transfer the amount from the contract which the donor deposited while invoking this method.

```

import { context, PersistentUnorderedMap, u128, PersistentMap, ContractPromiseBatch } from "near-sdk-as";

export function donate(ngoId: u32, projectId: u32):string {

  assert(ngoList.contains(ngoId), "NGO not found");

  assert(ngoProjectMap.getSome(ngoId).includes(projectId), "Project not found");

  const project = projects.getSome(projectId);

  assert(project.funds >= context.attachedDeposit, "So kind of you, but we do not accept excessive donations");

  

  const to_beneficiary = ContractPromiseBatch.create(project.address);

  project.funds = u128.sub(project.funds, context.attachedDeposit);

  projects.set(project.id, project);

  to_beneficiary.transfer(context.attachedDeposit);

  return "Done";

}

```

Here `ContractPromiseBatch` is just a class provided by sdk which provides us the ability to transfer NEAR tokens from deposited amounts to the project’s funding address using the `transfer()` function. In our case `ContractPromiseBatch.create(project.address).transfer(depositAmount)`

Don’t worry if all this seems alien to you, you will get more clarity on this once we call this `donate` function in the next subquest.
## Creating a NEAR account to send money to NGOs
Till now all of our transactions were executed using a development account which was generated by `near dev-deploy` command which generated a random account on the test network. But you might be wondering how one can create an account with a nice name in it just like `prtk.testnet`. Let’s get you one then.

Run below command on your console:

`near login`

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/aeedbe86-6e64-4311-86a5-bd2a1a52647a.jpg)

This will open up your web browser for generating a wallet. Just follow the instructions and you will be able to create your own wallet.

For reference check out below gif:

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/f7b0fd71-3eba-4de5-98c1-841a78589c58.jpg)

The credentials for this account will also be stored at `~/.near-credentials/testnet`

In order to deploy the smart contract to your account that you just created. You can use below command:

`near deploy --accountId example-contract.testnet --wasmFile out/example.wasm`

We just replace dev-deploy with deploy since we will not use a development account and pass the accountId (that you just created) where the contract is going to be deployed.
## Deployment and interaction with the contract
For our application to be publicly available, we will have to store it on the blockchain network, much like how web2 applications are hosted on a server like google cloud or aws.

First of all let’s compile the contract to WebAssembly file using:

`yarn asb`

Above command will create a .wasm file in the `build` directory.

Now let’s deploy using:

`near dev-deploy --wasmFile build/release/`

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/3aec9344-f11f-4191-8860-3f8cbb569cf3.jpg)

`near dev-deploy` looks for Near account credentials in your local machine and if found uses it to make the deployment, or else it creates an account in testnet and stores the credentials at path `~/.near-credentials/testnet/.json` for future use and then deploys using that newly created account. Keep your account id handy as we will use it for future calls.

To verify that the contract is deployed you may check out the near account using this link - [https://explorer.testnet.near.org/accounts/](https://explorer.testnet.near.org/accounts/)\[Account id\]  
Ex: https://explorer.testnet.near.org/accounts/dev-1630359371975-32794514962235

You should be able to see the contract like below screenshot:

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/3dd588f3-5867-4ed2-865c-2abfcab49bca.jpg)

Next, lets create a NGO:

```

near call  registerNGO '{"ngoName": "ngo1"}' --accountId  --gas=200000000000000

```

Ex:

`near call dev-1630359371975-32794514962235 registerNGO '{"ngoName": "ngo1"}' --accountId dev-1630359371975-32794514962235 --gas=200000000000000`

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/61f1694c-41c2-4ed6-962b-1fe9657a2a98.jpg)

Let’s dissect the command and understand what it does:

`near call` is used to invoke a function from a smart contract residing in the blockchain, now for the command to identify which contract it should interact with. Every contract is identified by the account ID. We pass the contract name (account id of deployer) and function name (`registerNGO`) with parameters the function is expecting (ngoName). We also need to say which account is invoking the function or interacting with the contract, we pass invoker’s account id using --accountId flag (your account id) with that we will provide a gas fee so that our transaction goes through and the storage on blockchain is modified using --gas flag.

Now we will allocate a project to this NGO:

`near call  addProject '{"ngoId": 0, "address": "prtk.testnet", "name": "COVID Vaccination drive", "funds": "10000000000000000000000000"}' --accountId  --gas=200000000000000`

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/972d24aa-2f77-4ba0-b878-cb6c67f53d56.jpg)

Now before donating funds to this project, let’s checkout balance of fund address [here](https://explorer.testnet.near.org/accounts/prtk.testnet):

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/e888d4b0-a72c-409e-b076-07d4f137d3d7.jpg)

And balance of donor account [here](https://explorer.testnet.near.org/accounts/dev-1630359371975-32794514962235):

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/3dce918d-6b08-4bba-b50f-35c6f55ae6ca.jpg)

Now let's make the donation:

`near call  donate '{"ngoId": 0, "projectId": 0}'  --accountId  --gas=200000000000000 --deposit=10`

Here  is invoker/donor and 10 NEAR tokens would be deposited from this account to our smart contract residing at  with the help of --deposit flag, which in turn would be transferred to the project's funding address as per the logic of `donate` function.

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/a5f9f0d1-3a69-43f9-a0e5-327f3f2264ea.jpg)

Checkout the updated balances now:

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/bd2487f2-261b-4284-8182-e94adac9431d.jpg)

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/27e20baa-71b0-42bc-a600-6ee241c9541c.jpg)

We successfully made the donation!
## What next?
In the current smart contract, there is a flaw which I want you to fix as a challenge:

Can you restrict the addition of projects only for NGO owner? Try implementing this and play around with the contract for a bit.