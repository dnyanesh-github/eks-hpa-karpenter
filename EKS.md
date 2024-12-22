---


---

<blockquote>
<p>Written with <a href="https://stackedit.io/">StackEdit</a>.</p>
</blockquote>
<h1 id="build-you-own-eks-cluster-with-autoscalinghpakarpenter">Build you own EKS cluster with autoscaling(HPA+Karpenter)</h1>
<p>“The grand finale is a wallet-saving move—skip it, and your wallet might file for bankruptcy!” - Don’t get it? Let me rephrase. If you care for your money, remember to execute the last step in the blog(Deleting the cluster!).</p>
<p>DON’T blame me, for your carelessness.</p>
<p>Alright, let’s get started.</p>
<p>“Welcome, dear reader, to the ultimate guide on deploying an EKS cluster with the magic wand of Kubernetes wizards—eksctl! If you’ve ever looked at cloud computing and thought, ‘Wow, this feels like assembling IKEA furniture without instructions,’ don’t worry—you’re not alone. But by the end of this blog, you’ll be deploying like a pro, and who knows, you might even find it fun (or at least less painful than putting together that bookshelf).”</p>
<p>Step 1: Installing AWS CLI</p>
<p>“First things first, download and install the AWS CLI. Think of it as your universal remote for the AWS world. Without it, you’re like a wizard without a wand—tragic and completely useless. Head to <a href="https://aws.amazon.com/cli/">AWS CLI installation guide</a> and follow the instructions. WAIT!</p>
<p>You know what, I read your mind. You don’t want to browse away and get that feeling what I always got when something isn’t thorough.</p>
<p>Let me provide the instructions here(Only for my dear Linux users though 😊).</p>
 
<h5>Install awsclia</h5>
<div>
    <pre id="content">curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
    </pre>
    
</div>


