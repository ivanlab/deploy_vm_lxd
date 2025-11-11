# Export
To export an Azure VHD for VMware, first, export the VHD from Azure using the Azure portal or PowerShell. Then, use a conversion tool like StarWind V2V Converter or qemu-img to convert the VHD file to the VMDK format. Finally, create a new virtual machine in VMware and attach the converted VMDK file. 

## Step 1: Export the VHD from Azure
Using the Azure Portal:
- Navigate to your Azure virtual machine and select "Disks".
- Choose the disk you want to export.
- Select "Disk Export" from the left-hand menu.
- Click "Generate URL" to create a shared access signature (SAS) URL for the VHD. Copy this URL.
- Use the URL to download the VHD file to your local machine.
  
Using Azure PowerShell:
- Open Azure PowerShell and use the Save-AzureVhd cmdlet.
- Provide the source URI of the VHD and the local file path where you want to save it.
- Example: Save-AzureVhd -Source "http://mystorage.blob.core.windows.net/vhds/myvm.vhd" -LocalFilePath "D:\exports\myvm.vhd". 

## Step 2: Convert VHD to VMDK
Using a tool like StarWind V2V Converter:
- Download and install the converter.
- Select the downloaded VHD file as the source.
- Choose "VMDK" as the destination format and select desired options like "Thin Provisioned".
- Start the conversion process.
  
Using qemu-img:
- Install qemu-img on your machine.
- Open a command-line interface and run the qemu-img convert command.
- Specify the input file (-f vpc or qcow2), the input file path, the output format (-O vmdk), and the output file path.
- Example: qemu-img convert -f vpc -O vmdk source.vhd target.vmdk. 

## Step 3: Import into VMware 
Open VMware Workstation or another VMware product.
- Create a new virtual machine.
- During the creation wizard, when prompted for a hard disk, select the option to use an existing virtual hard disk.
- Browse to and select your newly converted .vmdk file.
- Complete the wizard and start the virtual machine