# Installation and use
## Download 
First it's essential to get the source code. You can clone the code directly from the git repository using command
\begin{verbatim}
#git clone https://github.com/vojtsek/VideoCompression.git
\end{verbatim}
Alternatively you can download the zip file and unpack it in some directory.

## Requirements, installation and first run
To run the program successfully it's essential to have \textit{ffmpeg} and \textit{ffprobe} installed on your computer. Although technically it doesn't matter which codec is used, the program currently uses only H.264 standard for encoding of the output, so it assumes the \textit{ffmpeg} has been compiled with the \textit{x264} codec. Otherwise the program does not have any special requirements except standard libraries which should be available on all UNIX systems, so once you installed these programs, you can change to the directory containing the source code and run the installation script:
\begin{verbatim}
#cd DVC
#./install.sh
\end{verbatim}
The installation script is a regular Bash script, so the Bourne again shell interpreter is required to run it successfully. It explores your computer, i.e. gets the IP address, finds location of \textit{ffmpeg} binaries etc. Then it creates home directory for the program. The home directory contains data of the program's run. These include intermediate results as well as the final result and log files. The installation continues with generating the configuration file. This file is crucial for the framework. Before setting each option, the script prompts you for confirmation of the value. If you type nothing and just press the Enter key, the suggested value is used, otherwise the script uses your input. The resulting file is stored in the \textit{bin} directory which is created during the installation. It's a plain text file so you can edit it anytime in the future. The script then continues with compilation of the program. If everything is all right, you can change to the newly created \textit{bin} directory and continue. The directory \textit{bin/lists} contains some supporting files that should not be change, otherwise the program could behave improperly.

The last step before you can run the program is to check the configuration. The configuration is saved in the \textit{bin/CONF} file, which is created by the installation script. It's important to provide valid path to the \textit{ffmpeg} and \textit{ffprobe} executable and address with port of the neighbor that should be contacted initially. Otherwise you won't be able to join the network. The field *MY_IP* is not essential as long as the initial neighbor is alive - it will be recognized automatically. 

Then you can finally run the program. Some of the settings can be changed by providing options, the available ones are listed below. \pagebreak

 * -s ... no address will be contacted initially
 * -n address~port_number ... node to contact initially
 * -a address~port_number ... address and port to bound to
 * -h directory ... path to the home directory
 * -i file ... file to encode
 * -p port ... listening port
 * -d level ... debug level
 * -q quality ... quality of encoding

If the string *ipv4* appears among parameters, the program will use only the IPv4 addresses,
in which case should be the *CONF* file changed appropriately.

## Using the program
When you run the program, the initial screen appears. You can choose desired action then using function keys. You can see the initial screen in the figure below. Available options are highlighted.
\begin{figure}[h]
\begin{center}
\includegraphics[scale=0.35]{./img/init-screen.pdf}
\label{initial-screen}
\caption[initial-screen]{Initial screen after joining.}
\end{center}
\end{figure}

The important key bindings are listed in the table below.
\begin{table}[h]
\begin{center}
 \begin{tabular}{ | l | c |}
   \hline
   F6 & Show information \\ \hline
   F7 & Start the process \\ \hline
   F8 & Load the file \\ \hline
   F9 & Set values \\ \hline
   F10 & Abort the process \\ \hline
   F12 & Quit the program \\ \hline
   Up, Down & Traverse available options \\ \hline
   Enter & Confirm the input \\
 	\hline
 \end{tabular}
 \caption{Table of the control keys.}
 \end{center}
\end{table}

First you should load the video file. When the corresponding function key is pressed, the program prompts you for the file location. You can type in the absolute path of the file. Once you use the file, it is stored in history which you can browse using up and down arrow keys. 
\begin{figure}[h]
\begin{center}
\includegraphics[scale=0.35]{./img/loading.pdf}
\label{loading-files}
\caption{Loading the file.}
\end{center}
\end{figure}

Then you can set some parameters or show different information using F6 respectively F9 function keys. These keys provides set of options which you can choose from. When you are satisfied with the settings, you can start the process.
The program then starts splitting the file and distributing the chunks while informing you about the progress. When the process is done, the file is joined and you can do further actions.
\begin{figure}[h]
\begin{center}
\includegraphics[scale=0.35]{./img/process-initiator.pdf}
\caption{Overview of the process.}
\end{center}
\end{figure}
\begin{figure}[h]
\begin{center}
\includegraphics[scale=0.35]{./img/processing.pdf}
\caption{Processing of the tasks.}
\end{center}
\end{figure}
\begin{figure}[h]
\begin{center}
\includegraphics[scale=0.35]{./img/joining.pdf}
\caption{Joining the file.}
\end{center}
\end{figure}

To obtain more information about what is going on, the \textit{-d} option may be used which allows to specify level of debug messages that will be shown.