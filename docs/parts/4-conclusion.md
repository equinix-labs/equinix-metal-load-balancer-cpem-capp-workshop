<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->

# Part 4: Conclusion

Thank you for participating in the workshop! Let's recap some of the key takeways that we've learned:

- We learned how to create a Kubernetes cluster using Cluster API Provider Packet.
- We learned how to configure Cloud Provider Equinix Metal to set up service load balancers for us.
- We learned how to deploy a sample application to our Kubernetes cluster.

## Next Steps

- You may wish to delete your kubernetes cluster to avoid incurring any costs. You can do this by running the following command:

  ```shell
  export KUBECONFIG=kubeconfig-kind
  kubectl delete cluster my-lbaas-demo
  ```

  Then verify that the machines are deleted from your project.

- You can also delete the entire project. This will delete all resources in the project, including the machines and the project itself.

## Resources

Here are a few other resources to look at to continue your Equinix Metal journey:

- [Deploy @ Equinix](https://deploy.equinix.com): A one-stop shop for blogs, guides, and plenty of other resources.
- [Equinix Metal Docs](https://deploy.equinix.com/developers/docs/metal): Equinix Metal official documentation.
- [Equinix Metal APIs](https://deploy.equinix.com/developers/api/metal): Programmatically interact with Equinix Metal
- [Equinix Labs](https://github.com/equinix-labs): Provides SDKs and Terraform modultes for Infrastructure as Code tools.
- [Equinix Community](https://community.equinix.com): A global community for customers and Equinix users.
